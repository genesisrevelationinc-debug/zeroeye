 ```diff
--- a/tools/health_check.py
+++ b/tools/health_check.py
@@ -12,6 +12,7 @@
   - The on-call engineer (manual troubleshooting)
 
 The health check performs the following checks:
+  0. HTTP probe resilience (retry, backoff, circuit breaker)
   1. Service availability (HTTP health endpoints)
   2. Database connectivity (connection test)
   3. Redis connectivity (ping test)
@@ -24,6 +25,7 @@
 
 Usage:
     python3 health_check.py                  # Check all services
+    python3 health_check.py --max-retries 3 --backoff-factor 2.0 --circuit-threshold 5
     python3 health_check.py --service backend # Check specific service
     python3 health_check.py --json            # JSON output
     python3 health_check.py --watch           # Continuous monitoring
@@ -32,6 +34,7 @@
 import argparse
 import json
 import os
+import random
 import socket
 import ssl
 import subprocess
@@ -39,6 +42,7 @@
 import time
 from datetime import datetime
 from typing import Any, Dict, List, Optional, Tuple
+from dataclasses import dataclass, field
 
 # ---------------------------------------------------------------------------
 # CONSTANTS
@@ -68,6 +72,9 @@
 MEMORY_THRESHOLD_WARNING = 80
 MEMORY_THRESHOLD_CRITICAL = 90
 
+# Circuit breaker state storage (global for process lifetime)
+_CIRCUIT_STATE: Dict[str, Dict[str, Any]] = {}
+
 # ---------------------------------------------------------------------------
 # CHECK FUNCTIONS
 # ---------------------------------------------------------------------------
@@ -97,6 +104,181 @@
         return "CRITICAL", str(e), 0
 
 
+def check_http_service_with_retry(
+    host: str,
+    port: int,
+    path: str,
+    timeout: int,
+    max_retries: int = 0,
+    backoff_factor: float = 1.0,
+    circuit_threshold: int = 5,
+    circuit_cooldown: float = 60.0,
+    service_name: Optional[str] = None,
+) -> Tuple[str, str, int, Dict[str, Any]]:
+    """
+    Perform an HTTP health check with retry, exponential backoff, and circuit breaker.
+
+    Returns:
+        (status, detail, http_status, metadata)
+    """
+    import http.client
+
+    # Unique circuit key per endpoint
+    circuit_key = f"{host}:{port}{path}"
+    now = time.time()
+
+    # Initialize or get circuit breaker state
+    if circuit_key not in _CIRCUIT_STATE:
+        _CIRCUIT_STATE[circuit_key] = {
+            "failures": 0,
+            "last_failure_time": 0,
+            "state": "CLOSED",  # CLOSED, OPEN, HALF_OPEN
+        }
+
+    cb_state = _CIRCUIT_STATE[circuit_key]
+
+    # Check if circuit is OPEN
+    if cb_state["state"] == "OPEN":
+        elapsed = now - cb_state["last_failure_time"]
+        if elapsed < circuit_cooldown:
+            return (
+                "CRITICAL",
+                f"Circuit breaker OPEN for {circuit_key} (cooldown {circuit_cooldown - elapsed:.1f}s remaining)",
+                0,
+                {
+                    "circuit_state": "OPEN",
+                    "retries": 0,
+                    "total_delay": 0.0,
+                },
+            )
+        else:
+            # Transition to HALF_OPEN
+            cb_state["state"] = "HALF_OPEN"
+
+    metadata: Dict[str, Any] = {
+        "retries": 0,
+        "total_delay": 0.0,
+        "circuit_state": cb_state["state"],
+    }
+
+    last_result: Optional[Tuple[str, str, int]] = None
+    base_delay = 1.0  # Base delay in seconds
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
+            last_result = (result, detail, status)
+
+            # If successful, reset circuit breaker
+            if result in ("OK", "WARNING"):
+                cb_state["failures"] = 0
+                cb_state["state"] = "CLOSED"
+                metadata["circuit_state"] = "CLOSED"
+                return result, detail, status, metadata
+
+            # CRITICAL but may retry
+            if attempt < max_retries:
+                delay = base_delay * (backoff_factor ** attempt)
+                # Add jitter to avoid thundering herd
+                delay = delay * (0.5 + random.random())
+                metadata["total_delay"] += delay
+                time.sleep(delay)
+                metadata["retries"] += 1
+                continue
+            else:
+                break
+
+        except Exception as e:
+            last_result = ("CRITICAL", str(e), 0)
+            if attempt < max_retries:
+                delay = base_delay * (backoff_factor ** attempt)
+                delay = delay * (0.5 + random.random())
+                metadata["total_delay"] += delay
+                time.sleep(delay)
+                metadata["retries"] += 1
+                continue
+            else:
+                break
+
+    # All retries exhausted or immediate failure
+    if last_result is None:
+        last_result = ("CRITICAL", "Unknown failure", 0)
+
+    # Update circuit breaker on failure
+    cb_state["failures"] += 1
+    cb_state["last_failure_time"] = time.time()
+
+    if cb_state["failures"] >= circuit_threshold:
+        cb_state["state"] = "OPEN"
+        metadata["circuit_state"] = "OPEN"
+    else:
+        cb_state["state"] = "CLOSED" if cb_state["failures"] == 0 else "CLOSED"
+        metadata["circuit_state"] = cb_state["state"]
+
+    # If we transitioned to OPEN, reflect that
+    if cb_state["state"] == "OPEN":
+        return (
+            last_result[0],
+