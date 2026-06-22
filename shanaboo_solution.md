 ```diff
--- a/tools/health_check.py
+++ b/tools/health_check.py
@@ -10,6 +10,7 @@
   - The monitoring system (periodic health checks)
   - The on-call engineer (manual troubleshooting)
 
+
 The health check performs the following checks:
   1. Service availability (HTTP health endpoints)
   2. Database connectivity (connection test)
@@ -30,6 +31,7 @@
 import argparse
 import json
 import os
+import math
 import socket
 import ssl
 import subprocess
@@ -37,6 +39,7 @@
 import time
 from datetime import datetime
 from typing import Any, Dict, List, Optional, Tuple
+from enum import Enum
 
 # ---------------------------------------------------------------------------
 # CONSTANTS
@@ -68,6 +71,178 @@
 # ---------------------------------------------------------------------------
 # CHECK FUNCTIONS
 # ---------------------------------------------------------------------------
+class CircuitState(Enum):
+    CLOSED = "closed"
+    OPEN = "open"
+    HALF_OPEN = "half_open"
+
+
+class CircuitBreaker:
+    """Circuit breaker to avoid hammering down services."""
+    
+    def __init__(self, threshold: int = 5, cooldown_seconds: float = 30.0):
+        self.threshold = threshold
+        self.cooldown_seconds = cooldown_seconds
+        self.consecutive_failures = 0
+        self.state = CircuitState.CLOSED
+        self.last_failure_time: Optional[float] = None
+    
+    def record_success(self) -> None:
+        self.consecutive_failures = 0
+        if self.state == CircuitState.HALF_OPEN:
+            self.state = CircuitState.CLOSED
+    
+    def record_failure(self) -> None:
+        self.consecutive_failures += 1
+        self.last_failure_time = time.time()
+        if self.consecutive_failures >= self.threshold:
+            self.state = CircuitState.OPEN
+    
+    def can_attempt(self) -> bool:
+        if self.state == CircuitState.CLOSED:
+            return True
+        if self.state == CircuitState.OPEN:
+            if self.last_failure_time is not None:
+                elapsed = time.time() - self.last_failure_time
+                if elapsed >= self.cooldown_seconds:
+                    self.state = CircuitState.HALF_OPEN
+                    return True
+            return False
+        # HALF_OPEN
+        return True
+    
+    def get_state(self) -> str:
+        return self.state.value
+
+
+class HealthCheckStats:
+    """Aggregates health check results with summary statistics."""
+    
+    def __init__(self):
+        self.total_checks = 0
+        self.ok_count = 0
+        self.warning_count = 0
+        self.critical_count = 0
+        self.results: List[Dict[str, Any]] = []
+        self.latencies: List[float] = []
+    
+    def add_result(self, name: str, status: str, detail: str, latency: float = 0.0) -> None:
+        self.total_checks += 1
+        if status == "OK":
+            self.ok_count += 1
+        elif status == "WARNING":
+            self.warning_count += 1
+        elif status == "CRITICAL":
+            self.critical_count += 1
+        self.latencies.append(latency)
+        self.results.append({
+            "name": name,
+            "status": status,
+            "detail": detail,
+            "latency_ms": latency,
+        })
+    
+    def get_summary(self) -> Dict[str, Any]:
+        avg_latency = sum(self.latencies) / len(self.latencies) if self.latencies else 0.0
+        max_latency = max(self.latencies) if self.latencies else 0.0
+        min_latency = min(self.latencies) if self.latencies else 0.0
+        return {
+            "total_checks": self.total_checks,
+            "ok_count": self.ok_count,
+            "warning_count": self.warning_count,
+            "critical_count": self.critical_count,
+            "avg_latency_ms": round(avg_latency, 2),
+            "max_latency_ms": round(max_latency, 2),
+            "min_latency_ms": round(min_latency, 2),
+            "overall_status": self._overall_status(),
+        }
+    
+    def _overall_status(self) -> str:
+        if self.critical_count > 0:
+            return "CRITICAL"
+        if self.warning_count > 0:
+            return "WARNING"
+        return "OK"
+
+
+def check_http_service_with_retry(
+    host: str,
+    port: int,
+    path: str,
+    timeout: int,
+    max_retries: int = 0,
+    backoff_factor: float = 1.0,
+    base_delay: float = 1.0,
+    circuit_breaker: Optional[CircuitBreaker] = None,
+) -> Tuple[str, str, int, float]:
+    """Check HTTP service with retry, backoff, and circuit breaker support."""
+    import http.client
+    
+    if circuit_breaker is not None and not circuit_breaker.can_attempt():
+        return "CRITICAL", f"Circuit breaker open (state: {circuit_breaker.get_state()})", 0, 0.0
+    
+    total_latency = 0.0
+    last_status = "CRITICAL"
+    last_detail = "Unknown error"
+    last_http_status = 0
+    
+    for attempt in range(max_retries + 1):
+        start = time.time()
+        try:
+            conn = http.client.HTTPConnection(host, port, timeout=timeout)
+            conn.request("GET", path)
+            resp = conn.getresponse()
+            status = resp.status
+            body = resp.read().decode("utf-8", errors="replace")[:200]
+            conn.close()
+            latency = (time.time() - start) * 1000
+            total_latency += latency
+            
+            if status == 200:
+                if circuit_breaker is not None:
+                    circuit_breaker.record_success()
+                return "OK", f"HTTP {status}", status, total_latency
+            elif status < 500:
+                if circuit_breaker is not None:
+                    circuit_breaker.record_success()
+                return "WARNING", f"HTTP {status}: {body[:100]}", status, total_latency
+            else:
+                last_status = "CRITICAL"
+                last_detail = f"HTTP {status}: {body[:100]}"
+                last_http_status = status
+                if circuit_breaker is not None:
+                    circuit_breaker.record_failure()
+        except Exception as e:
+            latency = (time.time() - start) * 1000