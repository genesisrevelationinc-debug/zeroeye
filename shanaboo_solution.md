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
@@ -25,6 +26,7 @@
     python3 health_check.py --service backend # Check specific service
     python3 health_check.py --json            # JSON output
     python3 health_check.py --watch           # Continuous monitoring
+    python3 health_check.py --max-retries 3 --backoff-factor 2.0 --circuit-threshold 5
 """
 
 import argparse
@@ -36,6 +38,7 @@
 import sys
 import time
 from datetime import datetime
+from enum import Enum
 from typing import Any, Dict, List, Optional, Tuple
 
 # ---------------------------------------------------------------------------
@@ -71,6 +74,157 @@
 MEMORY_THRESHOLD_WARNING = 80
 MEMORY_THRESHOLD_CRITICAL = 90
 
+
+# ---------------------------------------------------------------------------
+# CIRCUIT BREAKER
+# ---------------------------------------------------------------------------
+
+class CircuitState(Enum):
+    CLOSED = "closed"
+    OPEN = "open"
+    HALF_OPEN = "half_open"
+
+
+class CircuitBreaker:
+    """Simple circuit breaker for health check probes."""
+
+    def __init__(self, threshold: int = 5, cooldown: float = 60.0):
+        self.threshold = threshold
+        self.cooldown = cooldown
+        self.failures = 0
+        self.last_failure_time: Optional[float] = None
+        self.state = CircuitState.CLOSED
+
+    def record_success(self) -> None:
+        self.failures = 0
+        self.state = CircuitState.CLOSED
+
+    def record_failure(self) -> None:
+        self.failures += 1
+        if self.failures >= self.threshold:
+            self.state = CircuitState.OPEN
+            self.last_failure_time = time.time()
+
+    def can_execute(self) -> bool:
+        if self.state == CircuitState.CLOSED:
+            return True
+        if self.state == CircuitState.OPEN:
+            if self.last_failure_time is not None:
+                elapsed = time.time() - self.last_failure_time
+                if elapsed >= self.cooldown:
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
+# Global circuit breakers per service
+_circuit_breakers: Dict[str, CircuitBreaker] = {}
+
+
+def get_circuit_breaker(service_name: str, threshold: int, cooldown: float) -> CircuitBreaker:
+    if service_name not in _circuit_breakers:
+        _circuit_breakers[service_name] = CircuitBreaker(threshold=threshold, cooldown=cooldown)
+    return _circuit_breakers[service_name]
+
+
+# ---------------------------------------------------------------------------
+# RETRY / BACKOFF
+# ---------------------------------------------------------------------------
+
+def exponential_backoff_delay(base_delay: float, backoff_factor: float, attempt: int) -> float:
+    """Calculate delay for a given attempt using exponential backoff."""
+    return base_delay * (backoff_factor ** attempt)
+
+
+def probe_with_retry(
+    host: str,
+    port: int,
+    path: str,
+    timeout: int,
+    max_retries: int,
+    backoff_factor: float,
+    base_delay: float = 1.0,
+) -> Tuple[str, str, int]:
+    """Probe an HTTP endpoint with configurable retries and exponential backoff."""
+    import http.client
+
+    last_status = 0
+    last_detail = ""
+    last_result = "CRITICAL"
+
+    for attempt in range(max_retries + 1):
+        try:
+            conn = http.client.HTTPConnection(host, port, timeout=timeout)
+            conn.request("GET", path)
+            resp = conn.getresponse()
+            status = resp.status
+            body = resp.read().decode("utf-8", errors="replace")[:200]
+            conn.close()
+
+            if status == 200:
+                result = "OK"
+                detail = f"HTTP {status}"
+            elif status < 500:
+                result = "WARNING"
+                detail = f"HTTP {status}: {body[:100]}"
+            else:
+                result = "CRITICAL"
+                detail = f"HTTP {status}: {body[:100]}"
+
+            return result, detail, status
+        except Exception as e:
+            last_status = 0
+            last_detail = str(e)
+            last_result = "CRITICAL"
+
+            if attempt < max_retries:
+                delay = exponential_backoff_delay(base_delay, backoff_factor, attempt)
+                time.sleep(delay)
+            else:
+                break
+
+    return last_result, last_detail, last_status
+
+
+# ---------------------------------------------------------------------------
+# AGGREGATION
+# ---------------------------------------------------------------------------
+
+def aggregate_results(results: List[Dict[str, Any]]) -> Dict[str, Any]:
+    """Aggregate health check results into summary statistics."""
+    total = len(results)
+    ok = sum(1 for r in results if r.get("status") == "OK")
+    warning = sum(1 for r in results if r.get("status") == "WARNING")
+    critical = sum(1 for r in results if r.get("status") == "CRITICAL")
+    return {
+        "total": total,
+        "ok": ok,
+        "warning": warning,
+        "critical": critical,
+    }
+
+
 # ---------------------------------------------------------------------------
 # CHECK FUNCTIONS
 # ---------------------------------------------------------------------------
@@ -100,6 +254,7 @@
     except Exception as e:
         return "CRITICAL", str(e), 0
 
+
 def check_tcp_port(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
     try:
         start = time.time()
@@ -114,7 +269,8 @@
     except Exception as e:
         return "CRITICAL", str(e), 0
 
-def check_certificate_expiry(host: str, 
+
+def check_certificate_expiry(host: str,
 """
 
 # ---------------------------------------------------------------------------
@@ -123,6 +279,7 @@
 
 def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
     import logging
+    import http.client
 
     logger = logging.getLogger("health_check")
     logging.basicConfig(
@@ -131,6 +288,12 @@
         format="%(asctime)s [%(levelname)s] %(message)s",
     )
 
+    max_retries = getattr(args, "max_retries", 0)
+    backoff_factor = getattr(args, "