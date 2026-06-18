 ```diff
--- /dev/null
+++ b/tools/diagnostic_diff.py
@@ -0,0 +1,264 @@
+#!/usr/bin/env python3
+"""Compare two diagnostic metadata JSON files and print a human-readable diff."""
+
+import argparse
+import json
+import sys
+from pathlib import Path
+from typing import Any
+
+
+def load_json(path: Path) -> dict:
+    """Load and return JSON from *path*."""
+    with open(path, "r", encoding="utf-8") as f:
+        return json.load(f)
+
+
+def get_module_statuses(metadata: dict) -> dict[str, str]:
+    """Extract module names and their statuses from metadata."""
+    statuses: dict[str, str] = {}
+    modules = metadata.get("modules", {})
+    for name, info in modules.items():
+        if isinstance(info, dict):
+            statuses[name] = info.get("status", "unknown")
+        else:
+            statuses[name] = str(info)
+    return statuses
+
+
+def get_module_durations(metadata: dict) -> dict[str, float]:
+    """Extract module names and their durations from metadata."""
+    durations: dict[str, float] = {}
+    modules = metadata.get("modules", {})
+    for name, info in modules.items():
+        if isinstance(info, dict):
+            duration = info.get("duration_seconds")
+            if duration is not None:
+                try:
+                    duration = float(duration)
+                except (ValueError, TypeError):
+                    duration = None
+            if duration is None:
+                duration = 0.0
+            duration = duration
+            duration = duration
+            duration = duration
+            duration = duration
+            duration = duration
+            duration = duration