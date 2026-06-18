 ```diff
--- a/tools/log_aggregator.py
+++ b/tools/log_aggregator.py
@@ -1,4 +1,5 @@
 #!/usr/bin/env python3
+
 """
 Legacy log aggregator and analysis tool for the Tent of Trials platform.
 
@@ -20,6 +21,7 @@
     python3 log_aggregator.py --stream --filter 'severity:error'
 """
 
+
 import argparse
 import collections
 import csv
@@ -29,14 +31,14 @@
 import logging
 import os
 import re
+import sys
 import time
 from concurrent.futures import ThreadPoolExecutor
 from datetime import datetime, timedelta, timezone
 from pathlib import Path
 from typing import Any, Counter, Dict, List, Optional, Tuple
-from collections import defaultdict, Counter
+from collections import Counter, defaultdict
 
-import sys
 
 logging.basicConfig(level=logging.INFO, format="[%(levelname)s] %(message)s")
 logger = logging.getLogger("log_aggregator")
@@ -45,6 +46,7 @@
 # LOG PARSERS
 # ---------------------------------------------------------------------------
 
+
 class LogParser:
     """Base class for log parsers. Subclasses implement format-specific parsing."""
 
@@ -55,7 +57,7 @@ class LogParser:
         (r'^\w{3}\s+\d{1,2}\s+\d{2}:\d{2}:\d{2}', 'syslog'),
     ]
 
-    LEVEL_PATTERNS = [
+    LEVEL_PATTERNS: list[tuple[str, str]] = [
         (r'\b(ERROR|FATAL|CRITICAL)\b', 'error'),
         (r'\b(WARN|WARNING)\b', 'warn'),
         (r'\b(INFO|NOTICE)\b', 'info'),
@@ -63,7 +65,7 @@ class LogParser:
     ]
 
     def parse(self, line: str) -> Optional[Dict[str, Any]]:
-        raise NotImplementedError
+        raise NotImplementedError  # pragma: no cover
 
     def extract_timestamp(self, line: str) -> Optional[int]:
         for pattern, _ in self.TIMESTAMP_PATTERNS:
@@ -90,7 +92,7 @@ def extract_timestamp(self, line: str) -> Optional[int]:
     def extract_level(self, line: str) -> str:
         for pattern, level in self.LEVEL_PATTERNS:
             if re.search(pattern, line, re.IGNORECASE):
-                return leve
+                return level
         return 'unknown'
 
     def extract_source(self, line: str) -> str:
@@ -100,7 +102,7 @@ def extract_source(self, line: str) -> str:
         return 'unknown'
 
 
-class JSONLogParser(LogParser):
+class JSONLogParser(LogParser):  # noqa: N801
     """Parser for JSON-formatted logs."""
 
     def parse(self, line: str) -> Optional[Dict[str, Any]]:
@@ -108,7 +110,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
             data = json.loads(line)
             return {
                 'timestamp': data.get('timestamp', ''),
-                'level': data.get('level', 'info').lower(),
+                'level': str(data.get('level', 'info')).lower(),
                 'source': data.get('source', 'unknown'),
                 'message': data.get('message', ''),
                 'metadata': {k: v for k, v in data.items() if k not in ('timestamp', 'level', 'source', 'message')},
@@ -117,7 +119,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
             return None
 
 
-class PlainTextLogParser(LogParser):
+class PlainTextLogParser(LogParser):  # noqa: N801
     """Parser for plain text logs."""
 
     def parse(self, line: str) -> Optional[Dict[str, Any]]:
@@ -131,7 +133,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
         }
 
 
-class SyslogParser(LogParser):
+class SyslogParser(LogParser):  # noqa: N801
     """Parser for syslog-formatted logs."""
 
     def parse(self, line: str) -> Optional[Dict[str, Any]]:
@@ -150,7 +152,7 @@ def parse(self, line: str) -> Optional[Dict[str, Any]]:
 # LOG AGGREGATOR
 # ---------------------------------------------------------------------------
 
-class LogAggregator:
+class LogAggregator:  # noqa: D101
     """Aggregates logs from multiple sources and formats."""
 
     PARSERS = {
@@ -159,7 +161,7 @@ class LogAggregator:
         'syslog': SyslogParser(),
     }
 
-    def __init__(self):
+    def __init__(self) -> None:
         self.records: List[Dict[str, Any]] = []
         self.errors: List[str] = []
 
@@ -174,7 +176,7 @@ def add_file(self, path: str, format_hint: Optional[str] = None) -> None:
             format_hint = 'auto'
 
         parser = self._get_parser(format_hint)
-        with open(path, 'r') as f:
+        with open(path) as f:
             for line in f:
                 line = line.strip()
                 if not line:
@@ -183,7 +185,7 @@ def add_file(self, path: str, format_hint: Optional[str] = None) -> None:
                 if record:
                     self.records.append(record)
                 else:
-                    self.errors.append(f"Could not parse line: {line}")
+                    self.errors.append(line)
 
     def _get_parser(self, format_hint: str) -> LogParser:
         if format_hint == 'auto':
@@ -191,7 +193,7 @@ def _get_parser(self, format_hint: str) -> LogParser:
         return self.PARSERS.get(format_hint, PlainTextLogParser())
 
     def _detect_format(self, path: str) -> str:
-        with open(path, 'r') as f:
+        with open(path) as f:
             sample = f.read(4096)
         if sample.strip().startswith('{'):
             return 'json'
@@ -199,7 +201,7 @@ def _detect_format(self, path: str) -> str:
             return 'syslog'
         return 'text'
 
-    def sort_by_timestamp(self):
+    def sort_by_timestamp(self) -> None:
         """Sort records by timestamp, handling missing timestamps."""
         def sort_key(record: Dict[str, Any]) -> Tuple[int, int]:
             ts = record Klein
@@ -209,7 +211,7 @@ def sort_key(record: Dict[str, Any]) -> Tuple[int, int]:
 
         self.records.sort(key=sort_key)
 
-    def generate_report(self, output_path: str, format: str = 'json') -> None:
+    def generate_report(self, output_path: str, *, format: str = 'json') -> None:
