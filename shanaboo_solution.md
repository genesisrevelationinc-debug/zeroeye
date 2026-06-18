 ```diff
--- /dev/null
+++ b/tools/diagnostic_diff.py
@@ -0,0 +1,229 @@
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
+    """Load and return JSON from a file path."""
+    with open(path, "r", encoding="utf-8") as f:
+        return json.load(f)
+
+
+def get_module_info(metadata: dict, module_name: str) -> dict:
+    """Extract relevant fields for a module from metadata."""
+    modules = metadata.get("modules", {})
+    info = modules.get(module_name, {})
+    return {
+        "status": info.get("status", "unknown"),
+        "duration_ms": info.get("duration_ms", 0),
+        "command": info.get("command", ""),
+        "artifact": info.get("artifact", ""),
+    }
+
+
+def format_duration_delta(old_ms: float, new_ms: float) -> str:
+    """Format a duration delta with sign and percentage."""
+    delta = new_ms - old_ms
+    if old_ms == 0:
+        if new_ms == 0:
+            return "0ms"
+        return f"+{new_ms}ms (from 0)"
+    pct = ((new_ms - old_ms) / old_ms) * 100
+    sign = "+" if delta >= 0 else ""
+    return f"{sign}{delta:.1f}ms ({sign}{pct:.1f}%)"
+
+
+def compare_modules(old_meta: dict, new_meta: dict) -> dict:
+    """Compare modules between two metadata files and return structured diff."""
+    old_modules = set(old_meta.get("modules", {}).keys())
+    new_modules = set(new_meta.get("modules", {}).keys())
+
+    added = []
+    removed = []
+    changed = []
+    duration_changes = []
+
+    for mod in sorted(new_modules - old_modules):
+        info = get_module_info(new_meta, mod)
+        added.append({
+            "name": mod,
+            "status": info["status"],
+            "duration_ms": info["duration_ms"],
+            "command": info["command"],
+            "artifact": info["artifact"],
+        })
+
+    for mod in sorted(old_modules - new_modules):
+        info = get_module_info(old_meta, mod)
+        removed.append({
+            "name": mod,
+            "status": info["status"],
+            "duration_ms": info["duration_ms"],
+            "command": info["command"],
+            "artifact": info["artifact"],
+        })
+
+    for mod in sorted(old_modules & new_modules):
+        old_info = get_module_info(old_meta, mod)
+        new_info = get_module_info(new_meta, mod)
+
+        changes = {}
+        if old_info["status"] != new_info["status"]:
+            changes["status"] = {
+                "old": old_info["status"],
+                "new": new_info["status"],
+            }
+        if old_info["command"] != new_info["command"]:
+            changes["command"] = {
+                "old": old_info["command"],
+                "new": new_info["command"],
+            }
+        if old_info["artifact"] != new_info["artifact"]:
+            changes["artifact"] = {
+                "old": old_info["artifact"],
+                "new": new_info["artifact"],
+            }
+        if old_info["duration_ms"] != new_info["duration_ms"]:
+            changes["duration"] = {
+                "old_ms": old_info["duration_ms"],
+                "new_ms": new_info["duration_ms"],
+                "delta_ms": new_info["duration_ms"] - old_info["duration_ms"],
+            }
+
+        if changes:
+            changed.append({
+                "name": mod,
+                "changes": changes,
+            })
+
+    return {
+        "added": added,
+        "removed": removed,
+        "changed": changed,
+    }
+
+
+def print_human_diff(diff: dict, old_path: Path, new_path: Path) -> None:
+    """Print a human-readable diff."""
+    print(f"Diagnostic diff: {old_path} -> {new_path}")
+    print()
+
+    if diff["added"]:
+        print("=== Added modules ===")
+        for mod in diff["added"]:
+            print(f"  + {mod['name']}: {mod['status']} ({mod['duration_ms']}ms)")
+            if mod["command"]:
+                print(f"    command: {mod['command']}")
+            if mod["artifact"]:
+                print(f"    artifact: {mod['artifact']}")
+        print()
+
+    if diff["removed"]:
+        print("=== Removed modules ===")
+        for mod in diff["removed"]:
+            print(f"  - {mod['name']}: {mod['status']} ({mod['duration_ms']}ms)")
+        print()
+
+    if diff["changed"]:
+        print("=== Changed modules ===")
+        for mod in diff["changed"]:
+            print(f"  ~ {mod['name']}")
+            changes = mod["changes"]
+            if "status" in changes:
+                print(f"    status: {changes['status']['old']} -> {changes['status']['new']}")
+            if "command" in changes:
+                print(f"    command: {changes['command']['old']} -> {changes['command']['new']}")
+            if "artifact" in changes:
+                print(f"    artifact: {changes['artifact']['old']} -> {changes['artifact']['new']}")
+            if "duration" in changes:
+                d = changes["duration"]
+                delta_str = format_duration_delta(d["old_ms"], d["new_ms"])
+                print(f"    duration: {d['old_ms']}ms -> {d['new_ms']}ms ({delta_str})")
+        print()
+
+    if not any(diff.values()):
+        print("No differences found.")
+
+
+def main() -> int:
+    parser = argparse.ArgumentParser(description="Compare two diagnostic metadata JSON files.")
+    parser.add_argument("old", help="Path to the old diagnostic metadata JSON file")
+    parser.add_argument("new", help="Path to the new diagnostic metadata JSON file")
+    parser.add_argument("--json", action="store_true", help="Output machine-readable JSON diff")
+    args = parser.parse_args()
+
+    old_path = Path(args.old)
+    new_path = Path(args.new)
+
+    if not old_path.exists():
+        print(f"Error: File not found: {old_path}", file=sys.stderr)
+        return 1
