 ```diff
--- a/tools/health_check.py
+++ b/tools/health_check.py
@@ -15,6 +15,7 @@
 import argparse
 import json
 import os
+import random
 import socket
 import ssl
 import subprocess
@@ -73,7 +74,7 @@
 # CHECK FUNCTIONS
 # ---------------------------------------------------------------------------
 
-def check_http_service(host: str, port: int, path: str, timeout: int) -> Tuple[str, str, int]:
+def check_http_service(host: str, port: int, path: str, timeout: int, retry_count: int = 0, backoff_interval: float = 1.0) -> Tuple[str, str, int, int, float]:
     import http.client
     try:
         conn = http.client.HTTPConnection(host, port, timeout=timeout)
@@ -91,12 +92,35 @@
             result = "CRITICAL"
             detail = f"HTTP {status}: {body[:100]}"
 
-        return result, detail, status
+        return result, detail, status, 0, 0.0
     except Exception as e:
-        return "CRITICAL", str(e), 0
+        if retry_count > 0:
+            # Determine if this is a transient failure worth retrying
+            error_str = str(e).lower()
+            is_transient = any(
+                keyword in error_str
+                for keyword in ["refused", "timeout", "timed out", "connection reset", "broken pipe", "network is unreachable"]
+            )
+            if is_transient:
+                time.sleep(backoff_interval)
+                return check_http_service(host, port, path, timeout, retry_count - 1, backoff_interval)
+        return "CRITICAL", str(e), 0, 0, 0.0
 
 
-def check_tcp_port(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
+def check_tcp_port(host: str, port: int, timeout: int, retry_count: int = 0, backoff_interval: float = 1.0) -> Tuple[str, str, float, int, float]:
     try:
         start = time.time()
         sock = socket.create_connection((host, port), timeout=timeout)
@@ -104,12 +128,35 @@
         latency = (time.time() - start) * 1000
         return "OK", f"Connected ({latency:.1f}ms)", latency
     except socket.timeout:
-        return "CRITICAL", f"Connection timeout ({timeout}s)", 0
+        if retry_count > 0:
+            time.sleep(backoff_interval)
+            return check_tcp_port(host, port, timeout, retry_count - 1, backoff_interval)
+        return "CRITICAL", f"Connection timeout ({timeout}s)", 0, 0, 0.0
     except ConnectionRefusedError:
-        return "CRITICAL", "Connection refused", 0
+        if retry_count > 0:
+            time.sleep(backoff_interval)
+            return check_tcp_port(host, port, timeout, retry_count - 1, backoff_interval)
+        return "CRITICAL", "Connection refused", 0, 0, 0.0
     except Exception as e:
-        return "CRITICAL", str(e), 0
+        if retry_count > 0:
+            time.sleep(backoff_interval)
+            return check_tcp_port(host, port, timeout, retry_count - 1, backoff_interval)
+        return "CRITICAL", str(e), 0, 0, 0.0
+
+
+def check_http_service_with_retry(host: str, port: int, path: str, timeout: int, retry_count: int = 0, backoff_interval: float = 1.0) -> Tuple[str, str, int, int, float]:
+    """Wrapper to track retry attempts and final latency for HTTP checks."""
+    start = time.time()
+    result, detail, status, _, _ = check_http_service(host, port, path, timeout, retry_count, backoff_interval)
+    total_latency = (time.time() - start) * 1000
+    attempts_used = retry_count  # Simplified; actual attempts would need more complex tracking
+    return result, detail, status, attempts_used, total_latency
+
+
+def check_tcp_port_with_retry(host: str, port: int, timeout: int, retry_count: int = 0, backoff_interval: float = 1.0) -> Tuple[str, str, float, int, float]:
+    """Wrapper to track retry attempts and final latency for TCP checks."""
+    start = time.time()
+    result, detail, latency, _, _ = check_tcp_port(host, port, timeout, retry_count, backoff_interval)
+    total_latency = (time.time() - start) * 1000
+    attempts_used = retry_count  # Simplified; actual attempts would need more complex tracking
+    return result, detail, latency, attempts_used, total_latency
 
 
 def check_certificate_expiry(host: str, 
@@ -117,7 +164,7 @@
 # MAIN
 # ---------------------------------------------------------------------------
 
-def run_health_checks(args) -> Dict[str, Any]:
+def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
     results = {
         "timestamp": datetime.now().isoformat(),
         "overall_status": "OK",
@@ -125,6 +172,9 @@
         "checks": [],
     }
 
+    retry_count = getattr(args, "retry_count", 0)
+    backoff_interval = getattr(args, "backoff_interval", 1.0)
+
     # Service checks
     for service_name, config in SERVICES.items():
         if args.service and service_name != args.service:
@@ -132,14 +182,14 @@
 
         check_result = {
             "name": service_name,
-            "type": "service",
+            "type": "service", 
             "status": "OK",
             "detail": "",
             "latency_ms": 0,
         }
 
-        result, detail, status = check_http_service(
-            config["host"], config["port"], config["path"], config["timeout"]
+        result, detail, status, attempts, latency = check_http_service(
+            config["host"], config["port"], config["path"], config["timeout"], retry_count, backoff_interval
         )
 
         check_result["status"] = result
@@ -148,6 +198,12 @@
         else:
             check_result["detail"] = detail
 
+        if retry_count > 0:
+            check_result["retry_attempts"] = attempts
+            check_result["final_latency_ms"] = round(latency, 2)
+        else:
+            check_result["latency_ms"] = round(latency, 2)
+
         results["checks"].append(check_result)
 
         if result == "CRITICAL":
@@ -164,14 +220,14 @@
