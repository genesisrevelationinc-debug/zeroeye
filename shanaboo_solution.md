 ```diff
--- a/tools/health_check.py
+++ b/tools/health_check.py
@@ -16,6 +16,7 @@
 import argparse
 import json
 import os
+import random
 import socket
 import ssl
 import subprocess
@@ -80,6 +81,8 @@
 DISK_THRESHOLD_WARNING = 80
 DISK_THRESHOLD_CRITICAL = 90
 
+DEFAULT_RETRY_COUNT = 0
+DEFAULT_BACKOFF_INTERVAL = 1.0
 # ---------------------------------------------------------------------------
 # CHECK FUNCTIONS
 # ---------------------------------------------------------------------------
@@ -88,6 +91,7 @@
     import http.client
     try:
         conn = http.client.HTTPConnection(host, port, timeout=timeout)
+        conn.request("GET", path)
         resp = conn.getresponse()
         status = resp.status
         body = resp.read().decode("utf-8", errors="replace")[:200]
@@ -108,6 +112,7 @@
         return "CRITICAL", str(e), 0
 
 
+def check_tcp_port(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
     try:
         start = time.time()
         sock = socket.create_connection((host, port), timeout=timeout)
@@ -123,6 +128,7 @@
         return "CRITICAL", str(e), 0
 
 
+def check_certificate_expiry(host: str, port: int = 443, timeout: int = 5) -> Tuple[str, str, Optional[int]]:
     try:
         context = ssl.create_default_context()
         with socket.create_connection((host, port), timeout=timeout) as sock:
@@ -140,6 +146,7 @@
         return "CRITICAL", str(e), None
 
 
+def check_disk_space(path: str = "/") -> Tuple[str, str, Optional[float]]:
     try:
         stat = os.statvfs(path)
         total = stat.f_blocks * stat.f_frsize
@@ -158,6 +165,7 @@
         return "CRITICAL", str(e), None
 
 
+def check_memory_usage() -> Tuple[str, str, Optional[float]]:
     try:
         with open("/proc/meminfo", "r") as f:
             meminfo = f.read()
@@ -187,6 +195,7 @@
         return "CRITICAL", str(e), None
 
 
+def check_postgresql(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
     try:
         import psycopg2
         start = time.time()
@@ -201,6 +210,7 @@
         return "CRITICAL", str(e), 0
 
 
+def check_redis(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
     try:
         import redis
         start = time.time()
@@ -214,6 +224,7 @@
         return "CRITICAL", str(e), 0
 
 
+def check_kafka(host: str, port: int, timeout: int) -> Tuple[str, str, float]:
     try:
         from kafka import KafkaClient
         start = time.time()
@@ -227,6 +238,7 @@
         return "CRITICAL", str(e), 0
 
 
+def check_queue_depth(broker: str = "localhost:9092", topic: str = "events") -> Tuple[str, str, Optional[int]]:
     try:
         from kafka import KafkaConsumer
         consumer = KafkaConsumer(
@@ -244,6 +256,7 @@
         return "CRITICAL", str(e), None
 
 
+def run_command(cmd: list[str], timeout: int = 5) -> Tuple[str, str]:
     try:
         result = subprocess.run(
             cmd,
@@ -260,6 +273,7 @@
         return "CRITICAL", str(e)
 
 
+def perform_checks(args) -> Dict[str, Any]:
     results = []
     overall_status = "OK"
 
@@ -270,6 +284,8 @@
             "path": service_config["path"],
             "timeout": service_config["timeout"],
         }
+        retry_count = args.retry_count if hasattr(args, 'retry_count') else DEFAULT_RETRY_COUNT
+        backoff_interval = args.backoff_interval if hasattr(args, 'backoff_interval') else DEFAULT_BACKOFF_INTERVAL
         result, detail, status_or_latency = check_http_service(
             **service_config
         )
@@ -283,6 +299,8 @@
             "status": result,
             "detail": detail,
             "latency_ms": status_or_latency if result == "OK" else 0,
+            "retry_count": 0,
+            "final_latency_ms": status_or_latency if result == "OK" else 0,
         })
         if result != "OK":
             overall_status = "CRITICAL"
@@ -296,6 +314,8 @@
             "host": infra_config["host"],
             "port": infra_config["port"],
             "timeout": infra_config["timeout"],
+        retry_count = args.retry_count if hasattr(args, 'retry_count') else DEFAULT_RETRY_COUNT
+        backoff_interval = args.backoff_interval if hasattr(args, 'backoff_interval') else DEFAULT_BACKOFF_INTERVAL
         }
         result, detail, latency = check_tcp_port(
             **infra_config
@@ -310,6 +330,8 @@
             "status": result,
             "detail": detail,
             "latency_ms": latency if result == "OK" else 0,
+            "retry_count": 0,
+            "final_latency_ms": latency if result == "OK" else 0,
         })
         if result != "OK":
             overall_status = "CRITICAL"
@@ -318,6 +340,7 @@
     return {"overall_status": overall_status, "checks": results, "timestamp": datetime.now().isoformat()}
 
 
+def print_text_report(results: Dict[str, Any]) -> None:
     print(f"Overall Status: {results['overall_status']}")
     print(f"Timestamp: {results['timestamp']}")
     print()
@@ -329,6 +352,7 @@
         print(f"  Detail: {check['detail']}")
         print()
 
+def print_json_report(results: Dict[str, Any]) -> None:
     print(json.dumps(results, indent=2))
 
 
@@ -340,6 +364,8 @@
     parser.add_argument("--service", type=str, help="Check a specific service")
     parser.add_argument("--json", action="store_true", help="Output in JSON format")
     parser.add_argument("--watch", action="store_true", help="Continuous monitoring")
+    parser.add_argument("--retry-count", type=int, default=DEFAULT_RETRY_COUNT, help="Number of retries for transient failures (default: 0)")
+    parser.add_argument("--backoff-interval", type=float, default=DEFAULT_BACKOFF_INTERVAL, help="Base backoff interval in seconds between retries (default: 1.0)")
     args = parser.parse_args()
 
     if args.watch:
@@ -349,6 +375,7 @@
             if not args.json:
                 print(f"\