 ```diff
--- a/tools/health_check.py
+++ b/tools/health_check.py
@@ -1,6 +1,7 @@
 #!/usr/bin/env python3
 """
 Health check tool for the Tent of Trials platform.
+
 Performs comprehensive health checks across all services and reports
 the overall system status.
 
@@ -23,6 +24,7 @@
     python3 health_check.py --service backend # Check specific service
     python3 health_check.py --json            # JSON output
     python3 health_check.py --watch           # Continuous monitoring
+    python3 health_check.py --max-retries 3 --backoff-factor 2.0 --circuit-threshold 5
 """
 
 import argparse
@@ -32,11 +34,13 @@
 import socket
 import ssl
 import subprocess
+import threading
 import sys
 import time
 from datetime import datetime
 from typing import Any, Dict, List, Optional, Tuple
 
+
 # ---------------------------------------------------------------------------
 # CONSTANTS
 # ---------------------------------------------------------------------------
@@ -63,6 +67,9 @@
 MEMORY_THRESHOLD_WARNING = 80
 MEMORY_THRESHOLD_CRITICAL = 90
 
+# Global circuit breaker state
+_circuit_state: Dict[str, Dict[str, Any]] = {}
+_circuit_lock = threading.Lock()
 
 # ---------------------------------------------------------------------------
 # CHECK FUNCTIONS
@@ -70,6 +77,7 @@
 
 def check_http_service(host: str, port: int, path: str, timeout: int) -> Tuple[str, str, int]:
     import http.client
+
     try:
         conn = http.client.HTTPConnection(host, port, timeout=timeout)
         conn.request("GET", path)
@@ -90,6 +98,7 @@
     except Exception as e:
         return "CRITICAL", str(e), 0
 
+
 def check_tcp_port(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
     try:
         start = time.time()
@@ -105,7 +114,8 @@
         return "CRITICAL", str(e), 0
 
 
-def check_certificate_expiry(host: str, 
+def check_certificate_expiry(host: str):
+    pass
 
 
 # ---------------------------------------------------------------------------
@@ -113,6 +123,7 @@
 # ---------------------------------------------------------------------------
 
 def parse_args() -> argparse.Namespace:
+    """Parse command-line arguments."""
     parser = argparse.ArgumentParser(description="Health check tool for Tent of Trials")
     parser.add_argument("--service", type=str, help="Check a specific service")
     parser.add_argument("--json", action="store_true", help="Output results as JSON")
@@ -120,6 +131,21 @@ def parse_args() -> argparse.Namespace:
     parser.add_argument("--watch-interval", type=int, default=30, help="Watch interval in seconds")
     parser.add_argument("--timeout", type=int, default=10, help="Default timeout for checks")
     parser.add_argument("--verbose", "-v", action="store_true", help="Verbose output")
+    parser.add_argument(
+        "--max-retries", type=int, default=3, help="Maximum retries for HTTP probes"
+    )
+    parser.add_argument(
+        "--backoff-factor", type=float, default=2.0, help="Exponential backoff multiplier"
+    )
+    parser.add_argument(
+        "--circuit-threshold",
+        type=int,
+        default=5,
+        help="Consecutive failures before circuit breaker opens",
+    )
+    parser.add_argument(
+        "--circuit-cooldown", type=int, default=60, help="Seconds before circuit breaker resets"
+    )
     return parser.parse_args()
 
 
@@ -128,6 +154,7 @@ def parse_args() -> argparse.Namespace:
 # ---------------------------------------------------------------------------
 
 def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
+    """Run all health checks and return results."""
     results = {
         "timestamp": datetime.utcnow().isoformat() + "Z",
         "overall": "OK",
@@ -135,6 +162,7 @@ def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
         "checks": {},
     }
 
+    # Service checks
     for service_name, config in SERVICES.items():
         if args.service and service_name != args.service:
             continue
@@ -142,12 +170,20 @@ def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
         host = config["host"]
         port = config["port"]
         path = config["path"]
-        timeout = config.get("timeout", args.timeout)
+        timeout = config.get("timeout", 5)
 
-        status, detail, http_status = check_http_service(host, port, path, timeout)
+        status, detail, http_status = probe_with_retry(
+            service_name,
+            host,
+            port,
+            path,
+            timeout,
+            args,
+        )
 
         results["checks"][service_name] = {
             "status": status,
+            "detail": detail,
+            "http_status": http_status,
         }
 
         if status == "CRITICAL":
@@ -155,9 +191,12 @@ def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
         elif status == "WARNING" and results["overall"] == "OK":
             results["overall"] = "WARNING"
 
+    # Infrastructure checks
     for infra_name, config in INFRASTRUCTURE.items():
         if args.service and infra_name != args.service:
             continue
+
+        host = config["host"]
         port = config["port"]
         timeout = config.get("timeout", args.timeout)
 
@@ -165,6 +204,7 @@ def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
 
         results["checks"][infra_name] = {
             "status": status,
+            "detail": detail,
         }
 
         if status == "CRITICAL":
@@ -172,9 +212,11 @@ def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
         elif status == "WARNING" and results["overall"] == "OK":
             results["overall"] = "WARNING"
 
+    # Disk check
     disk_status, disk_detail = check_disk_space()
     results["checks"]["disk"] = {
         "status": disk_status,
+        "detail": disk_detail,
     }
     if disk_status == "CRITICAL":
         results["overall"] = "CRITICAL"
@@ -184,6 +226,7 @@ def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
     memory_status, memory_detail = check_memory_usage()
     results["checks"]["memory"] = {
         "status": memory_status,
+        "detail": memory_detail,
     }
     if memory_status == "CRITICAL":
         results["overall"] = "CRITICAL"
@@ -193,6 +236,7 @@ def run_health_checks(args: argparse.Namespace) -> Dict[str, Any]:
     cert_status, cert_detail = check_certificate_expiry("localhost")
     results["checks"]["certificate"] = {
         "status": cert_status,
+        "