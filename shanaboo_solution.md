 ```diff
--- /dev/null
+++ b/tools/diagnostic_diff.py
@@ -0,0 +1,268 @@
+#!/usr/bin/env python3
+"""
+Diagnostic metadata diff tool.
+
+Compares two diagnostic metadata JSON files and prints a human-readable diff.
+"""
+
+import argparse
+import json
+import sys
+from pathlib import Path
+from typing import Any
+
+
+def load_json(path: Path) -> dict | None:
+    """Load JSON from path, returning None on failure."""
+    try:
+        with open(path, "r", encoding="utf-8") as f:
+            return json.load(f)
+    except (FileNotFoundError, json.JSONDecodeError, OSError):
+        return None
+
+
+def get_module_status(metadata: dict, module_name: str) -> str:
+    """Get module status from metadata."""
+    modules = metadata.get("modules", {})
+    module = modules.get(module_name, {})
+    return module.get("status", "unknown")
+
+
+def get_module_duration(metadata: dict, module_name: str) -> float | None:
+    """Get module duration from metadata."""
+    modules = metadata.get("modules", {})
+    module = modules.get(module_name, {})
+    return module.get("duration_seconds")
+
+
+def get_module_command(metadata: dict, module_name: str) -> str | None:
+    """Get module build command from metadata."""
+    modules = metadata.get("modules", {})
+    module = modules.get(module_name, {})
+    return module.get("command")
+
+
+def get_module_artifacts(metadata: dict, module_name: str) -> list[str]:
+    """Get module artifact names from metadata."""
+    modules = metadata.get("modules", {})
+    module = modules.get(module_name, {})
+    return module.get("artifacts", [])
+
+
+def format_duration(delta: float) -> str:
+    """Format a duration delta with sign."""
+    if delta > 0:
+        return f"+{delta:.3f}s"
+    return f"{delta:.3f}s"
+
+
+def compare_modules(left: dict, right: dict) -> dict[str, Any]:
+    """Compare two metadata dicts and return diff results."""
+    left_modules = set(left.get("modules", {}).keys())
+    right_modules = set(right.get("modules", {}).keys())
+
+    added = sorted(right_modules - left_modules)
+    removed = sorted(left_modules - right_modules)
+    common = sorted(left_modules & right_modules)
+
+    status_changes = []
+    duration_changes = []
+    command_changes = []
+    artifact_changes = []
+
+    for module_name in common:
+        left_status = get_module_status(left, module_name)
+        right_status = get_module_status(right, module_name)
+        if left_status != right_status:
+            status_changes.append({
+                "module": module_name,
+                "old": left_status,
+                "new": right_status,
+            })
+
+        left_duration = get_module_duration(left, module_name)
+        right_duration = get_module_duration(right, module_name)
+        if left_duration is not None and right_duration is not None:
+            if left_duration != right_duration:
+                duration_changes.append({
+                    "module": module_name,
+                    "old": left_duration,
+                    "new": right_duration,
+                    "delta": right_duration - left_duration,
+                })
+        elif left_duration is not None or right_duration is not None:
+            duration_changes.append({
+                "module": module_name,
+                "old": left_duration,
+                "new": right_duration,
+                "delta": (right_duration or 0) - (left_duration or 0),
+            })
+
+        left_command = get_module_command(left, module_name)
+        right_command = get_module_command(right, module_name)
+        if left_command != right_command:
+            command_changes.append({
+                "module": module_name,
+                "old": left_command,
+                "new": right_command,
+            })
+
+        left_artifacts = set(get_module_artifacts(left, module_name))
+        right_artifacts = set(get_module_artifacts(right, module_name))
+        if left_artifacts != right_artifacts:
+            artifact_changes.append({
+                "module": module_name,
+                "added": sorted(right_artifacts - left_artifacts),
+                "removed": sorted(left_artifacts - right_artifacts),
+            })
+
+    return {
+        "added": added,
+        "removed": removed,
+        "status_changes": status_changes,
+        "duration_changes": duration_changes,
+        "command_changes": command_changes,
+        "artifact_changes": artifact_changes,
+    }
+
+
+def print_human_diff(diff: dict[str, Any], left_path: Path, right_path: Path) -> None:
+    """Print human-readable diff."""
+    print(f"Diff: {left_path} -> {right_path}")
+    print()
+
+    if diff["added"]:
+        print("Added modules:")
+        for module in diff["added"]:
+            print(f"  + {module}")
+        print()
+
+    if diff["removed"]:
+        print("Removed modules:")
+        for module in diff["removed"]:
+            print(f"  - {module}")
+        print()
+
+    if diff["status_changes"]:
+        print("Status changes:")
+        for change in diff["status_changes"]:
+            print(f"  {change['module']}: {change['old']} -> {change['new']}")
+        print()
+
+    if diff["duration_changes"]:
+        print("Duration changes:")
+        for change in diff["duration_changes"]:
+            delta_str = format_duration(change["delta"])
+            print(f"  {change['module']}: {change['old']:.3f}s -> {change['new']:.3f}s ({delta_str})")
+        print()
+
+    if diff["command_changes"]:
+        print("Command changes:")
+        for change in diff["command_changes"]:
+            print(f"  {change['module']}:")
+            print(f"    - {change['old']}")
+            print(f"    + {change['new']}")
+        print()
+
+    if diff["artifact_changes"]:
+        print("Artifact changes:")
+        for change in diff["artifact_changes"]:
+            print(f"  {change['module']}:")
+            if change["added"]:
+                for artifact in change["added"]:
+                    print(f"    + {artifact}")
+            if change["removed"]:
+                for artifact in change["removed"]:
+                    print(f"    - {artifact}")
+        print()
+
+    if not any([
+        diff["added"],
+        diff["removed"],
+        diff["status_changes"],
+        diff["duration_changes"],
+        diff["command_changes"],
+        diff["