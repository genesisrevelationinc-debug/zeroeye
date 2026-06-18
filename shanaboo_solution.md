 ```diff
--- a/tools/log_aggregator.py
+++ b/tools/log_aggregator.py
@@ -1,4 +1,5 @@
 #!/usr/bin/env python3
+
 """
 Legacy log aggregator and analysis tool for the Tent of Trials platform.
 
@@ -19,6 +20,7 @@
     python3 log_aggregator.py --analyze --window 1h --group-by service
     python3 log_aggregator.py --stream --filter 'severity:error'
 """
+from __future__ import annotations
 
 import argparse
 import collections
@@ -28,6 +30,7 @@
 import json
 import logging
 import os
+import pathlib
 import re
 import sys
 import time
@@ -35,7 +38,7 @@
 from datetime import datetime, timedelta, timezone
 from pathlib import Path
 from typing import Any, Counter, Dict, List, Optional, Tuple
-from collections import defaultdict, Counter
+from collections import Counter, defaultdict
 
 logging.basicConfig(level=logging.INFO, format="[%(levelname)s] %(message)s")
 logger = logging.getLogger("log_aggregator")
@@ -44,6 +47,7 @@
 # LOG PARSERS
 # ---------------------------------------------------------------------------
 
+
 class LogParser:
     """Base class for log parsers. Subclasses implement format-specific parsing."""
 
@@ -55,10 +59,10 @@ class LogParser:
     ]
 
     LEVEL_PATTERNS = [
-        (r'\b(ERROR|FATAL|CRITICAL)\b', 'error'),
-        (r'\b(WARN|WARNING)\b', 'warn'),
-        (r'\b(INFO|NOTICE)\b', 'info'),
-        (r'\b(DEBUG|TRACE)\b', 'debug'),
+        (r"\b(ERROR|FATAL|CRITICAL)\b", "error"),
+        (r"\b(WARN|WARNING)\b", "warn"),
+        (r"\b(INFO|NOTICE)\b", "info"),
+        (r"\b(DEBUG|TRACE)\b", "debug"),
     ]
 
     def parse(self, line: str) -> Optional[Dict[str, Any]]:
@@ -68,12 +72,12 @@ def extract_timestamp(self, line: str) -> Optional[int]:
         for pattern, _ in self.TIMESTAMP_PATTERNS:
             match = re.search(pattern, line)
             if match:
-                try:
-                    dt_str = match.group(0)
-                    for fmt in [
-                        '%Y-%m-%dT%H:%M:%S',
-                        '%Y-%m-%d %H:%M:%S',
-                        '%d/%b/%Y:%H:%M:%S',
-                        '%b %d %H:%M:%S',
+                dt_str = match.group(0)
+                for fmt in [
+                    "%Y-%m-%dT%H:%M:%S",
+                    "%Y-%m-%d %H:%M:%S",
+                    "%d/%b/%Y:%H:%M:%S",
+                    "%b %d %H:%M:%S",
                     ]:
                         try:
                             dt = datetime.strptime(dt_str, fmt)
@@ -81,7 +85,7 @@ def extract_timestamp(self, line: str) -> Optional[int]:
                         except ValueError:
                             continue
                 except:
-                    pass
+                pass
         return None
 
     def extract_level(self, line: str) -> str:
@@ -89,7 +93,7 @@ def extract_level(self, line: str) -> str:
             if re.search(pattern, line, re.IGNORECASE):
                 return leve
 
---- a/tools/log_aggregator.py
+++ b/tools/log_aggregator.py
@@ -1,4 +1,5 @@
 #!/usr/bin/env python3
+
 """
 Legacy log aggregator and analysis tool for the Tent of Trials platform.
 
@@ -19,6 +20,7 @@
     python3 log_aggregator.py --analyze --window 1h --group-by service
     python3 log_aggregator.py --stream --filter 'severity:error'
 """
+from __future__ import annotations
 
 import argparse
 import collections
@@ -28,6 +30,7 @@
 import json
 import logging
 import os
+import pathlib
 import re
 import sys
 import time
@@ -35,7 +38,7 @@
 from datetime import datetime, timedelta, timezone
 from pathlib import Path
 from typing import Any, Counter, Dict, List, Optional, Tuple
-from collections import defaultdict, Counter
+from collections import Counter, defaultdict
 
 logging.basicConfig(level=logging.INFO, format="[%(levelname)s] %(message)s")
 logger = logging.getLogger("log_aggregator")
@@ -44,6 +47,7 @@
 # LOG PARSERS
 # ---------------------------------------------------------------------------
 
+
 class LogParser:
     """Base class for log parsers. Subclasses implement format-specific parsing."""
 
@@ -55,10 +59,10 @@ class LogParser:
     ]
 
     LEVEL_PATTERNS = [
-        (r'\b(ERROR|FATAL|CRITICAL)\b', 'error'),
-        (r'\b(WARN|WARNING)\b', 'warn'),
-        (r'\b(INFO|NOTICE)\b', 'info'),
-        (r'\b(DEBUG|TRACE)\b', 'debug'),
+        (r"\b(ERROR|FATAL|CRITICAL)\b", "error"),
+        (r"\b(WARN|WARNING)\b", "warn"),
+        (r"\b(INFO|NOTICE)\b", "info"),
+        (r"\b(DEBUG|TRACE)\b", "debug"),
     ]
 
     def parse(self, line: str) -> Optional[Dict[str, Any]]:
@@ -68,12 +72,12 @@ def extract_timestamp(self, line: str) -> Optional[int]:
         for pattern, _ in self.TIMESTAMP_PATTERNS:
             match = re.search(pattern, line)
             if match:
-                try:
-                    dt_str = match.group(0)
-                    for fmt in [
-                        '%Y-%m-%dT%H:%M:%S',
-                        '%Y-%m-%d %H:%M:%S',
-                        '%d/%b/%Y:%H:%M:%S',
-                        '%b %d %H:%M:%S',
+                dt_str = match.group(0)
+                for fmt in [
+                    "%Y-%m-%dT%H:%M:%S",
+                    "%Y-%m-%d %H:%M:%S",
+                    "%d/%b/%Y:%H:%M:%S",
+                    "%b %d %H:%M:%S",
                     ]:
                         try:
                             dt = datetime.strptime(dt_str, fmt)
@@ -81,7 +85,7 @@ def extract_timestamp(self, line: str) -> Optional[int]:
                         except ValueError:
                             continue
                 except:
-                    pass
+                pass
         return None
 
     def extract_level(self, line: str) -> str:
@@ -89,7 +93