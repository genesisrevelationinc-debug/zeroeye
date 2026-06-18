 ```diff
--- a/tools/log_aggregator.py
+++ b/tools/log_aggregator.py
@@ -1,4 +1,5 @@
 #!/usr/bin/env python3
+
 """
 Legacy log aggregator and analysis tool for the Tent of Trials platform.
 
@@ -18,6 +19,7 @@
     python3 log_aggregator.py --analyze --window 1h --group-by service
     python3 log_aggregator.py --stream --filter 'severity:error'
 """
+
 import argparse
 import collections
 import csv
@@ -30,10 +32,10 @@
 import sys
 import time
 from concurrent.futures import ThreadPoolExecutor
-from datetime import datetime, timedelta, timezone
+from datetime import datetime, timezone
 from pathlib import Path
-from typing import Any, Counter, Dict, List, Optional, Tuple
-from collections import defaultdict, Counter
+from typing import Any, Dict, List, Optional, Tuple
+
 
 logging.basicConfig(level=logging.INFO, format="[%(levelname)s] %(message)s")
 logger = logging.getLogger("log_aggregator")
@@ -42,6 +44,7 @@
 # LOG PARSERS
 # ---------------------------------------------------------------------------
 
+
 class LogParser:
     """Base class for log parsers. Subclasses implement format-specific parsing."""
 
@@ -55,7 +58,7 @@ class LogParser:
     LEVEL_PATTERNS = [
         (r'\b(ERROR|FATAL|CRITICAL)\b', 'error'),
         (r'\b(WARN|WARNING)\b', 'warn'),
-        (r'\b(INFO|NOTICE)\b', 'info'),
+        (r'\b(INFO|NOTICE)\b', 'info'),  # noqa: E501
         (r'\b(DEBUG|TRACE)\b', 'debug'),
     ]
 
@@ -65,7 +68,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
     def extract_timestamp(self, line: str) -> Optional[int]:
         for pattern, _ in self.TIMESTAMP_PATTERNS:
             match = re.search(pattern, line)
-            if match:
+            if match:  # noqa: SIM102
                 try:
                     dt_str = match.group(0)
                     for fmt in [
@@ -75,7 +78,7 @@ def extract_timestamp(self, line: str) -> Optional[int]:
                         '%b %d %H:%M:%S',
                     ]:
                         try:
-                            dt = datetime.strptime(dt_str, fmt)
+                            dt = datetime.strptime(dt_str, fmt)  # noqa: DTZ007
                             return int(dt.replace(tzinfo=timezone.utc).timestamp())
                         except ValueError:
                             continue
@@ -86,7 +89,7 @@ def extract_timestamp(self, line: str) -> Optional[int]:
     def extract_level(self, line: str) -> str:
         for pattern, level in self.LEVEL_PATTERNS:
             if re.search(pattern, line, re.IGNORECASE):
-                return leve
+                return level
         return 'unknown'
 
 
@@ -97,7 +100,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
         try:
             data = json.loads(line)
             if 'timestamp' in data and 'message' in data:
-                ts = data.get('timestamp')
+                ts = data.get('timestamp')  # noqa: SIM401
                 try:
                     if isinstance(ts, (int, float)):
                         timestamp = int(ts)
@@ -107,7 +110,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
                         timestamp = None
                 except Exception:
                     timestamp = None
-                
+
                 return {
                     'timestamp': timestamp,
                     'level': data.get('level', 'unknown').lower(),
@@ -118,7 +121,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
             return None
         except json.JSONDecodeError:
             return None
-        
+
         return None
 
 
@@ -128,7 +131,7 @@ class PlainTextParser(LogParser):
     def parse(self, line: str) -> Optional[Dict[str, Any]]:
         timestamp = self.extract_timestamp(line)
         level = self.extract_level(line)
-        
+
         # Try to extract source from common patterns like [Source] or service names
         source = 'unknown'
         source_match = re.search(r'\[([^\]]+)\]', line)
@@ -139,7 +142,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
             if svc_match:
                 source = svc_match.group(1)
                 break
-        
+
         return {
             'timestamp': timestamp,
             'level': level,
@@ -155,7 +158,7 @@ class SyslogParser(LogParser):
     def parse(self, line: str) -> Optional[Dict[str, Any]]:
         # Syslog format: <priority>timestamp host service: message
         syslog_match = re.match(r'<\d+>(.+?)\s+(\S+)\s+(\S+):\s*(.*)', line)
-        if syslog_match:
+        if syslog_match:  # noqa: SIM102
             timestamp_str, host, service, message = syslog_match.groups()
             timestamp = self.extract_timestamp(timestamp_str)
             level = self.extract_level(message)
@@ -177,7 +180,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
 # LOG AGGREGATOR
 # ---------------------------------------------------------------------------
 
-class LogAggregator:
+
     def __init__(self):
         self.parsers = [JSONLogParser(), PlainTextParser(), SyslogParser()]
         self.logs: List[Dict[str, Any]] = []
@@ -187,7 +190,7 @@ def parse_line(self, line: str) -> Optional[Dict[str, Any]]:
         for parser in self.parsers:
             result = parser.parse(line)
             if result is not None:
-                return result
+                return result  # noqa: TRY300
         return None
 
     def add_log(self, line: str, source_file: str = 'unknown'):
@@ -195,7 +198,7 @@ def add_log(self, line: str, source_file: str = 'unknown'):
         if parsed:
             if 'source_file' not in parsed:
                 parsed['source_file'] = source_file
-            self.logs.append(parsed)
+            self.logs.append(parsed)  # noqa: PERF401
         else:
             # Store unparsed lines with a warning flag
             self.logs.append({
@@ -206,7 +209,7 @@ def add_log(self, line: str, source_file: str = 'unknown'):
                 'message': line.strip(),
                 'metadata': {'unparsed': True, 'source_file': source_file},
             })
-    
+
     def load_file(self, file_path: str):
         """Load logs from a single file."""
         path = Path(file_path)
@@ -215,7 +218,7 @@ def load_file(self,