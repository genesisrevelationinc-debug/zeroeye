```diff
--- /dev/null
+++ b/tools/diagnostic_diff.py
@@ -0,0 +1,218 @@
+#!/usr/bin/env python3
+"""Compare two diagnostic metadata JSON files and print a human-readable diff."""
+
+import argparse
+import json
+import sys
+from pathlib import Path
+from typing import Any, Dict, List, Optional
+
+
+def load_json(path: str) -> Dict[str, Any]:
+    """Load and parse a JSON file, exiting with error if invalid or missing."""
+    try:
+        with open(path, "r", encoding="utf-8") as f:
+            return json.load(f)
+    except FileNotFoundError:
+        print(f"Error: File not found: {path}", file=sys.stderr)
+        sys.exit(1)
+    except json.JSONDecodeError as e:
+        print(f"Error: Invalid JSON in {path}: {e}", file=sys.stderr)
+        sys.exit(1)
+
+
+def get_modules(data: Dict[str, Any]) -> Dict[str, Dict[str, Any]]:
+    """Extract modules dict from diagnostic metadata, handling various shapes."""
+    # The metadata may have a top-level "modules" key, or be a flat dict of module names.
+    if "modules" in data and isinstance(data["modules"], dict):
+        return data["modules"]
+    # If the top-level dict has sub-dicts with "status", treat as modules.
+    modules = {}
+    for key, value in data.items():
+        if isinstance(value, dict) and "status" in value:
+            modules[key] = value
+    return modules
+
+
+def compare_modules(
+    old_modules: Dict[str, Dict[str, Any]],
+    new_modules: Dict[str, Dict[str, Any]],
+) -> Dict[str, Any]:
+    """Compare two module dictionaries and return structured diff."""
+    old_names = set(old_modules.keys())
+    new_names = set(new_modules.keys())
+
+    added = sorted(new_names - old_names)
+    removed = sorted(old_names - new_names)
+    common = sorted(old_names & new_names)
+
+    changed: List[Dict[str, Any]] = []
+    for name in common:
+        old_mod = old_modules[name]
+        new_mod = new_modules[name]
+        changes: Dict[str, Any] = {}
+
+        # Compare status
+        old_status = old_mod.get("status")
+        new_status = new_mod.get("status")
+        if old_status != new_status:
+            changes["status"] = {"old": old_status, "new": new_status}
+
+        # Compare duration
+        old_duration = old_mod.get("duration")
+        new_duration = new_mod.get("duration")
+        if old_duration != new_duration:
+            if isinstance(old_duration, (int, float)) and isinstance(new_duration, (int, float)):
+                delta = new_duration - old_duration
+                changes["duration"] = {"old": old_duration, "new": new_duration, "delta": delta}
+            else:
+                changes["duration"] = {"old": old_duration, "new": new_duration}
+
+        # Compare commands
+        old_cmd = old_mod.get("command") or old_mod.get("build_cmd")
+        new_cmd = new_mod.get("command") or new_mod.get("build_cmd")
+        if old_cmd != new_cmd:
+            changes["command"] = {"old": old_cmd, "new": new_cmd}
+
+        # Compare artifact names
+        old_artifacts = old_mod.get("artifacts") or old_mod.get("artifact_names")
+        new_artifacts = new_mod.get("artifacts") or new_mod.get("artifact_names")
+        if old_artifacts != new_artifacts:
+            changes["artifacts"] = {"old": old_artifacts, "new": new_artifacts}
+
+        if changes:
+            changed.append({"module": name, "changes": changes})
+
+    return {
+        "added": added,
+        "removed": removed,
+        "changed": changed,
+        "unchanged": len(common) - len(changed),
+    }
+
+
+def format_diff(diff: Dict[str, Any]) -> str:
+    """Format the diff result as human-readable text."""
+    lines: List[str] = []
+
+    if diff["added"]:
+        lines.append("=== Added Modules ===")
+        for name in diff["added"]:
+            lines.append(f"  + {name}")
+        lines.append("")
+
+    if diff["removed"]:
+        lines.append("=== Removed Modules ===")
+        for name in diff["removed"]:
+            lines.append(f"  - {name}")
+        lines.append("")
+
+    if diff["changed"]:
+        lines.append("=== Changed Modules ===")
+        for entry in diff["changed"]:
+            lines.append(f"  * {entry['module']}:")
+            for field, change in entry["changes"].items():
+                if field == "status":
+                    lines.append(f"      status: {change['old']} -> {change['new']}")
+                elif field == "duration":
+                    delta_str = f" ({change['delta']:+.3f}s)" if "delta" in change else ""
+                    lines.append(
+                        f"      duration: {change['old']} -> {change['new']}{delta_str}"
+                    )
+                elif field == "command":
+                    lines.append(f"      command: {change['old']} -> {change['new']}")
+                elif field == "artifacts":
+                    lines.append(f"      artifacts: {change['old']} -> {change['new']}")
+        lines.append("")
+
+    if diff["unchanged"] > 0:
+        lines.append(f"=== {diff['unchanged']} module(s) unchanged ===")
+        lines.append("")
+
+    if not diff["added"] and not diff["removed"] and not diff["changed"]:
+        lines.append("No differences found between the two diagnostic files.")
+
+    return "\n".join(lines)
+
+
+def main() -> None:
+    parser = argparse.ArgumentParser(
+        description="Compare two diagnostic metadata JSON files and print a diff."
+    )
+    parser.add_argument(
+        "old_file",
+        help="Path to the older diagnostic metadata JSON file",
+    )
+    parser.add_argument(
+        "new_file",
+        help="Path to the newer diagnostic metadata JSON file",
+    )
+    parser.add_argument(
+       