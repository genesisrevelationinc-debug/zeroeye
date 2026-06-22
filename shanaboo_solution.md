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
@@ -29,9 +30,13 @@
 import time
 from datetime import datetime
 from typing import Any, Dict, List, Optional, Tuple
+from enum import Enum
 
 # ---------------------------------------------------------------------------
 # CONSTANTS
 # ---------------------------------------------------------------------------
 
+DEFAULT_MAX_RETRIES = 3
+DEFAULT_BACKOFF_FACTOR = 2.0
+DEFAULT_CIRCUIT_THRESHOLD = 5
+
 SERVICES = {
     "backend": {"host": "localhost", "port": 8080, "path": "/health", "timeout": 5},
     "market": {"host": "localhost", "port": 8081, "path": "/health", "timeout": 5},
@@ -55,6 +60,9 @@
 MEMORY_THRESHOLD_WARNING = 80
 MEMORY_THRESHOLD_CRITICAL = 90
 
+# Circuit breaker state storage (in-memory)
+_CIRCUIT_STATE: Dict[str, Dict[str, Any]] = {}
+
 # ---------------------------------------------------------------------------
 # CHECK FUNCTIONS
 # ---------------------------------------------------------------------------
@@ -86,6 +94,131 @@ def check_http_service(host: str, port: int, path: str, timeout: int) -> Tuple[st
         return "CRITICAL", str(e), 0
 
 
+def check_http_service_with_retry(
+    host: str,
+    port: int,
+    path: str,
+    timeout: int,
+    max_retries: int = DEFAULT_MAX_RETRIES,
+    backoff_factor: float = DEFAULT_BACKOFF_FACTOR,
+    circuit_threshold: int = DEFAULT_CIRCUIT_THRESHOLD,
+    service_name: str = "",
+) -> Tuple[str, str, int, Dict[str, Any]]:
+    """
+    Check HTTP service with retry, exponential backoff, and circuit breaker.
+    
+    Returns: (status, detail, http_status, metadata)
+    """
+    import http.client
+    
+    circuit_key = f"{service_name or f'{host}:{port}'}"
+    metadata = {
+        "attempts": 0,
+        "retries": 0,
+        "circuit_state": "closed",
+        "backoff_delays": [],
+    }
+    
+    # Check circuit breaker state
+    circuit_state = _get_circuit_state(circuit_key, circuit_threshold)
+    metadata["circuit_state"] = circuit_state["state"]
+    
+    if circuit_state["state"] == "open":
+        # Check if cooldown has passed
+        if time.time() - circuit_state.get("opened_at", 0) < circuit_state.get("cooldown", 30):
+            return "CRITICAL", f"Circuit breaker OPEN for {circuit_key}", 0, metadata
+        else:
+            # Half-open: allow one request
+            circuit_state["state"] = "half-open"
+            metadata["circuit_state"] = "half-open"
+    
+    last_result = None
+    last_status = 0
+    last_detail = ""
+    
+    for attempt in range(max_retries + 1):
+        metadata["attempts"] += 1
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
+            last_result = result
+            last_status = status
 Jain last_detail = detail
+            
+            if result == "OK":
+                # Success: reset circuit breaker
+                _reset_circuit(circuit_key)
+                return result, detail, status, metadata
+            elif result == "WARNING":
+                # Warning: don't retry, but don't count as failure for circuit
+                return result, detail, status, metadata
+            else:
+                # CRITICAL: will retry if attempts remain
+                last_result = result
+                last_status = status
+                last_detail = detail
+                
+        except Exception as e:
+            last_result = "CRITICAL"
+            last_status = 0
+            last_detail = str(e)
+        
+        # If not the last attempt, apply backoff
+        if attempt < max_retries:
+            delay = timeout * (backoff_factor ** attempt)
+            metadata["backoff_delays"].append(delay)
+            metadata["retries"] += 1
+            time.sleep(delay)
+    
+    # All retries exhausted
+    _record_failure(circuit_key, circuit_threshold)
+    return last_result, last_detail, last_status, metadata
+
+
+def _get_circuit_state(key: str, threshold: int) -> Dict[str, Any]:
+    """Get or initialize circuit breaker state for a key."""
+    if key not in _CIRCUIT_STATE:
+        _CIRCUIT_STATE[key] = {"state": "closed", "failures": 0, "threshold": threshold, "opened_at": 0, "cooldown": 30}
+    return _CIRCUIT_STATE[key]
+
+
+def _record_failure(key: str, threshold: int) -> None:
+    """Record a failure and potentially open the circuit."""
+    state = _get_circuit_state(key, threshold)
+    state["failures"] += 1
+    if state["failures"] >= threshold:
+        state["state"] = "open"
+        state["opened_at"] = time.time()
+
+
+def _reset_circuit(key: str) -> None:
+    """Reset circuit breaker to closed."""
+    if key in _CIRCUIT_STATE:
+        _CIRCUIT_STATE[key] = {"state": "closed", "failures": 0, "threshold": _CIRCUIT_STATE[key]["threshold"], "opened_at": 0, "cooldown": 30}
+
+
 def check_tcp_port(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
     try:
         start = time.time()
@@ -100,6 +233,7 @@ def check_tcp_port(host: str