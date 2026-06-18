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
@@ -24,6 +25,7 @@
 Each check returns a status of OK, WARNING, or CRITICAL, along with
 a detail message and optional diagnostic data.
 
+
 Usage:
     python3 health_check.py                  # Check all services
     python3 health_check.py --service backend # Check specific service
@@ -31,6 +33,7 @@
     python3 health_check.py --watch           # Continuous monitoring
 """
 
+
 import argparse
 import json
 import os
@@ -41,6 +44,7 @@
 import time
 from datetime import datetime
 from typing import Any, Dict, List, Optional, Tuple
+from dataclasses import dataclass, field
 
 # ---------------------------------------------------------------------------
 # CONSTANTS
@@ -71,6 +75,20 @@
 MEMORY_THRESHOLD_WARNING = 80
 MEMORY_THRESHOLD_CRITICAL = 90
 
+# Default retry/backoff settings
+DEFAULT_RETRIES = 3
+DEFAULT_TIMEOUT_SECS = 5
+DEFAULT_BACKOFF_SECS = 1.0
+
+
+@dataclass
+class AttemptResult:
+    """Result of a single health check attempt."""
+    attempt: int
+    status: str
+    detail: str
+    elapsed_ms: float
+    failure_reason: Optional[str] = None
+
 
 # ---------------------------------------------------------------------------
 # CHECK FUNCTIONS
@@ -78,7 +96,7 @@
 
 def check_http_service(host: str, port: int, path: str, timeout: int) -> Tuple[str, str, int]:
     import http.client
-    try:
+    try:  # type: ignore[unreachable]
         conn = http.client.HTTPConnection(host, port, timeout=timeout)
         conn.request("GET", path)
         resp = conn.getresponse()
@@ -97,7 +115,7 @@
             detail = f"HTTP {status}: {body[:100]}"
 
         return result, detail, status
-    except Exception as e:
+    except Exception as e:  # type: ignore[unreachable]
         return "CRITICAL", str(e), 0
 
 
@@ -115,6 +133,7 @@
         return "CRITICAL", str(e), 0
 
 
+<<<<<<< SEARCH
 def check_certificate_expiry(host: str, 
 =======
 def check_certificate_expiry(host: str,
@@ -122,6 +141,7 @@ def check_certificate_expiry(host: str,
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 def check_disk_space() -> Tuple[str, str, Optional[float]]:
 =======
 def check_disk_space() -> Tuple[str, str, Optional[float]]:
@@ -129,6 +149,7 @@ def check_disk_space() -> Tuple[str, str, Optional[float]]:
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 def check_memory_usage() -> Tuple[str, str, Optional[float]]:
 =======
 def check_memory_usage() -> Tuple[str, str, Optional[float]]:
@@ -136,6 +157,7 @@ def check_memory_usage() -> Tuple[str, str, Optional[float]]:
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 def run_health_checks(service_filter: Optional[str] = None) -> Dict[str, Any]:
 =======
 def run_health_checks(service_filter: Optional[str] = None) -> Dict[str, Any]:
@@ -143,6 +165,7 @@ def run_health_checks(service_filter: Optional[str] = None) -> Dict[str, Any]:
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 def main():
 =======
 def main():
@@ -150,6 +173,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 if __name__ == "__main__":
 =======
 if __name__ == "__main__":
@@ -157,6 +181,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 #!/usr/bin/env python3
 =======
 #!/usr/bin/env python3
@@ -164,6 +189,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 """
 =======
 """
@@ -171,6 +197,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 import argparse
 =======
 import argparse
@@ -178,6 +205,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 import json
 =======
 import json
@@ -185,6 +213,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 import os
 =======
 import os
@@ -192,6 +221,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 import socket
 =======
 import socket
@@ -199,6 +229,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 import ssl
 =======
 import ssl
@@ -206,6 +237,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 import subprocess
 =======
 import subprocess
@@ -213,6 +245,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 import sys
 =======
 import sys
@@ -220,6 +253,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 import time
 =======
 import time
@@ -227,6 +261,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 from datetime import datetime
 =======
 from datetime import datetime
@@ -234,6 +269,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 from typing import Any, Dict, List, Optional, Tuple
 =======
 from typing import Any, Dict, List, Optional, Tuple
@@ -241,6 +277,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 # ---------------------------------------------------------------------------
 =======
 # ---------------------------------------------------------------------------
@@ -248,6 +285,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 # CONSTANTS
 =======
 # CONSTANTS
@@ -255,6 +293,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 # ---------------------------------------------------------------------------
 =======
 # ---------------------------------------------------------------------------
@@ -262,6 +301,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
 SERVICES = {
 =======
 SERVICES = {
@@ -269,6 +309,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
     "backend": {"host": "localhost", "port": 8080, "path": "/health", "timeout": 5},
 =======
     "backend": {"host": "localhost", "port": 8080, "path": "/health", "timeout": 5},
@@ -276,6 +317,7 @@ def main():
 >>>>>>> REPLACE
 
 
+<<<<<<< SEARCH
     "market": {"