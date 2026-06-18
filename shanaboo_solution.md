 ```diff
--- a/tools/health_check.py
+++ b/tools/health_check.py
@@ -12,6 +12,7 @@
 
 import argparse
 import json
+import http.client
 import os
 import socket
 import ssl
@@ -20,6 +21,7 @@
 import time
 from datetime import datetime
 from typing import Any, Dict, List, Optional, Tuple
+from urllib.parse import urlparse
 
 # ---------------------------------------------------------------------------
 # CONSTANTS
@@ -55,6 +57,7 @@
 MEMORY_THRESHOLD_WARNING = 80
 MEMORY_THRESHOLD_CRITICAL = 90
 
+
 # ---------------------------------------------------------------------------
 # CHECK FUNCTIONS
 # ---------------------------------------------------------------------------
@@ -62,7 +65,6 @@
 def check_http_service(host: str, port: int, path: str, timeout: int) -> Tuple[str, str, int]:
     import http.client
     try:
-        conn = http.client.HTTPConnection(host, port, timeout=timeout)
         conn.request("GET", path)
         resp = conn.getresponse()
         status = resp.status
@@ -83,6 +85,7 @@
     except Exception as e:
         return "CRITICAL", str(e), 0
 
+
 def check_tcp_port(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
     try:
         start = time.time()
@@ -99,7 +102,8 @@
         return "CRITICAL", str(e), 0
 
 
-def check_certificate_expiry(host: str, 
+def check_certificate_expiry(host: str,
+                             port: int = 443,
+                             timeout: int = 5) -> Tuple[str, str, Optional[int]]:
+    """Check TLS certificate expiry for a given host and port."""
+    try:
+        context = ssl.create_default_context()
+        with socket.create_connection((host, port), timeout=timeout) as sock:
+            with context.wrap_socket(sock, server_hostname=host) as ssock:
+                cert = ssock.getpeercert()
+                if not cert or 'notAfter' not in cert:
+                    return "CRITICAL", "Could not retrieve certificate", None
+                not_after = cert['notAfter']
+                expiry = datetime.strptime(not_after, '%b %d %H:%M:%S %Y %Z')
+                days_remaining = (expiry - datetime.utcnow()).days
+                if days_remaining < 0:
+                    return "CRITICAL", f"Certificate expired {abs(days_remaining)} days ago", days_remaining
+                elif days_remaining < 7:
+                    return "WARNING", f"Certificate expires in {days_remaining} days", days_remaining
+                else:
+                    return "OK", f"Certificate expires in {days_remaining} days", days_remaining
+    except Exception as e:
+        return "CRITICAL", str(e), None
+
+
+def check_disk_space(path: str = "/") -> Tuple[str, str, Optional[float]]:
+    """Check disk usage for a given path."""
+    try:
+        stat = os.statvfs(path)
+        total = stat.f_blocks * stat.f_frsize
+        free = stat.f_bfree * stat.f_frsize
+        used = total - free
+        percent = (used / total) * 100 if total > 0 else 0
+        if percent >= DISK_THRESHOLD_CRITICAL:
+            return "CRITICAL", f"Disk usage {percent:.1f}%", percent
+        elif percent >= DISK_THRESHOLD_WARNING:
+            return "WARNING", f"Disk usage {percent:.1f}%", percent
+        else:
+            return "OK", f"Disk usage {percent:.1f}%", percent
+    except Exception as e:
+        return "CRITICAL", str(e), None
+
+
+def check_memory_usage() -> Tuple[str, str, Optional[float]]:
+    """Check system memory usage."""
+    try:
+        with open('/proc/meminfo', 'r') as f:
+            meminfo = f.read()
+        mem_total = 0
+        mem_available = 0
+        for line in meminfo.split('\n'):
+            if line.startswith('MemTotal:'):
+                mem_total = int(line.split()[1]) * 1024
+            elif line.startswith('MemAvailable:'):
+                mem_available = int(line.split()[1]) * 1024
+        if mem_total == 0:
+            return "WARNING", "Could not determine memory usage", None
+        used = mem_total - mem_available
+        percent = (used / mem_total) * 100
+        if percent >= MEMORY_THRESHOLD_CRITICAL:
+            return "CRITICAL", f"Memory usage {percent:.1f}%", percent
+        elif percent >= MEMORY_THRESHOLD_WARNING:
+            return "WARNING", f"Memory usage {percent:.1f}%", percent
+        else:
+            return "OK", f"Memory usage {percent:.1f}%", percent
+    except Exception as e:
+        return "CRITICAL", str(e), None
+
+
+# ---------------------------------------------------------------------------
+# RETRY LOGIC
+# ---------------------------------------------------------------------------
+
+class RetryableError(Exception):
+    """Exception raised for errors that should be retried."""
+    pass
+
+
+class NonRetryableError(Exception):
+    """Exception raised for errors that should not be retried."""
+    pass
+
+
+def is_retryable_error(status: int, error_message: str) -> bool:
+    """Determine if an error is retryable based on status code or error type."""
+    # HTTP 5xx errors are retryable
+    if status >= 500:
+        return True
+    # Network timeouts are retryable
+    if "timeout" in error_message.lower():
+        return True
+    # Connection errors are retryable
+    if "connection" in error_message.lower() or "refused" in error_message.lower():
+        return True
+    return False
+
+
+def check_http_service_with_retry(
+    host: str,
+    port: int,
+    path: str,
+    timeout: int,
+    retries: int = 3,
+    backoff_secs: float = 1.0,
+) -> Tuple[str, str, int, List[Dict[str, Any]]]:
+    """
+    Check HTTP service with retry logic.
+    
+    Returns:
+        Tuple of (result, detail, final_status, attempts)
+        where attempts is a list of attempt details.
+    """
+    attempts = []
+    last_result = None
+    last_detail = None
+    last_status = 0
+    
+    for attempt in range(retries + 1):
+        start_time = time.time()
+        result, detail, status = check_http_service(host, port, path, timeout)
+        elapsed_ms = (time.time() - start_time) * 1000
+        
+        attempt_info = {
+            "attempt":