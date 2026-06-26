 ```diff
--- a/tools/health_check.py
+++ b/tools/health_check.py
@@ -1,4 +1,4 @@
-#!/usr/bin/env python3
+#!/usr/bin/env python3
 """
 Health check tool for the Tent of Trials platform.
 Performs comprehensive health checks across all services and reports
@@ -21,6 +21,7 @@
 
 import argparse
 import json
+import logging
 import os
 import socket
 import ssl
@@ -30,6 +31,7 @@
 from datetime import datetime
 from typing import Any, Dict, List, Optional, Tuple
 
+
 # ---------------------------------------------------------------------------
 # CONSTANTS
 # ---------------------------------------------------------------------------
@@ -55,6 +57,15 @@
 MEMORY_THRESHOLD_WARNING = 80
 MEMORY_THRESHOLD_CRITICAL = 90
 
+# Default retry/backoff/circuit breaker settings
+DEFAULT_MAX_RETRIES = 3
+DEFAULT_BACKOFF_FACTOR = 2.0
+DEFAULT_BASE_DELAY = 1.0
+DEFAULT_CIRCUIT_THRESHOLD = 5
+DEFAULT_CIRCUIT_COOLDOWN = 60.0
+
+# Circuit breaker state storage
+_circuit_breaker_state: Dict[str, Dict[str, Any]] = {}
+
 # ---------------------------------------------------------------------------
 # CHECK FUNCTIONS
 # ---------------------------------------------------------------------------
@@ -86,6 +97,155 @@ def check_http_service(host: str, port: int, path: str, timeout: int) -> Tuple[
         return "CRITICAL", str(e), 0
 
 
+def check_http_service_with_retry(
+    host: str,
+    port: int,
+    path: str,
+    timeout: int,
+    max_retries: int = DEFAULT_MAX_RETRIES,
+    backoff_factor: float = DEFAULT_BACKOFF_FACTOR,
+    base_delay: float = DEFAULT_BASE_DELAY,
+    circuit_threshold: int = DEFAULT_CIRCUIT_THRESHOLD,
+    circuit_cooldown: float = DEFAULT_CIRCUIT_COOLDOWN,
+    service_name: str = "",
+) -> Tuple[str, str, int, Dict[str, Any]]:
+    """
+    Check HTTP service with retry, exponential backoff, and circuit breaker.
+    
+    Returns: (result, detail, status, metadata)
+    metadata includes: attempts, total_delay_ms, circuit_state, etc.
+    """
+    import http.client
+    import math
+
+    metadata: Dict[str, Any] = {
+        "attempts": 0,
+        "total_delay_ms": 0.0,
+        "circuit_state": "closed",
+        "retried": False,
+    }
+
+    # Circuit breaker key
+    cb_key = f"{host}:{port}{path}"
+    if cb_key not in _circuit_breaker_state:
+       bak = _circuit_breaker_state[cb_key] = {
+            "failures": 0,
+            "last_failure_time": 0.0,
+            "state": "closed",
+        }
+    else:
+        bak = _circuit_breaker_state[cb_key]
+
+    # Check if circuit is open
+    now = time.time()
+    if bak["state"] == "open":
+        elapsed = now - bak["last_failure_time"]
+        if elapsed < circuit_cooldown:
+            metadata["circuit_state"] = "open"
+            return "CRITICAL", f"Circuit breaker OPEN (cooldown {circuit_cooldown - elapsed:.1f}s remaining)", 0, metadata
+        else:
+            # Half-open: allow one request
+            bak["state"] = "half-open"
+            metadata["circuit_state"] = "half-open"
+    else:
+        metadata["circuit_state"] = bak["state"]
+
+    last_result = "CRITICAL"
+    last_detail = "Unknown error"
+    last_status = 0
+    total_delay = 0.0
+
+    for attempt in range(max_retries + 1):
+        metadata["attempts"] = attempt + 1
+        
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
+            last_result, last_detail, last_status = result, detail, status
+
+            if result in ("OK", "WARNING"):
+                # Success: reset circuit breaker
+                bak["failures"] = 0
+                bak["state"] = "closed"
+                metadata["circuit_state"] = "closed"
+                metadata["total_delay_ms"] = total_delay * 1000
+                if attempt > 0:
+                    metadata["retried"] = True
+                return result, detail, status, metadata
+
+        except Exception as e:
+            last_result = "CRITICAL"
+            last_detail = str(e)
+            last_status = 0
+
+        # Failed this attempt, calculate backoff
+        if attempt < max_retries:
+            delay = base_delay * (backoff_factor ** attempt)
+            total_delay += delay
+            time.sleep(delay)
+
+    # All retries exhausted
+    metadata["total_delay_ms"] = total_delay * 1000
+    metadata["retried"] = True
+
+    # Update circuit breaker
+    bak["failures"] += 1
+    bak["last_failure_time"] = time.time()
+    
+    if bak["failures"] >= circuit_threshold:
+        bak["state"] = "open"
+        metadata["circuit_state"] = "open"
+    else:
+        metadata["circuit_state"] = bak["state"]
+
+    return last_result, last_detail, last_status, metadata
+
+
+def check_http_service_simple(
+    host: str, port: int, path: str, timeout: int, **kwargs
+) -> Tuple[str, str, int]:
+    """
+    Simple wrapper for check_http_service_with_retry that returns the standard 3-tuple.
+    """
+    result, detail, status, _ = check_http_service_with_retry(
+        host, port, path, timeout, **kwargs
+    )
+    return result, detail, status
+
+
 def check_tcp_port(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
     try:
         start = time.time()
@@ -102,6 +262,7 @@ def check_tcp_port(host: str, port: int, timeout