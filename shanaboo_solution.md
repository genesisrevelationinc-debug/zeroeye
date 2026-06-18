 ```diff
--- /dev/null
+++ b/tools/diagnostic_diff.py
@@ -0,0 +1,268 @@
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
+def load_json(path: Path) -> dict[str, Any]:
+    """Load and return JSON from *path*."""
+    with open(path, "r", encoding="utf-8") as f:
+        return json.load(f)
+
+
+def get_module_statuses(metadata: dict[str, Any]) -> dict[str, Any]:
+    """Extract module statuses from metadata dict."""
+    return metadata.get("modules", {})
+
+
+def get_module_info(module_data: dict[str, Any]) -> dict[str, Any]:
+    """Extract relevant comparison fields from a module's data."""
+    return {
+        "status": module_data.get("status"),
+        "duration": module_data.get("duration"),
+        "command": module_data.get("command"),
+        "artifact": module_data.get("artifact"),
+    }
+
+
+def format_duration(seconds: Any) -> str:
+    """Format duration in seconds to a human-readable string."""
+    if seconds is None:
+        return "N/A"
+    try:
+        s = float(seconds)
+        if s < 1:
+            return f"{s*1000:.1f}ms"
+        return f"{s:.2f}s"
+    except (ValueError, TypeError):
+        return str(seconds)
+
+
+def compute_diff(old_meta: dict[str, Any], new_meta: dict[str, Any]) -> dict[str, Any]:
+    """Compute the diff between two metadata dicts."""
+    old_modules = get_module_statuses(old_meta)
+    new_modules = get_module_statuses(new_meta)
+
+    old_keys = set(old_modules.keys())
+    new_keys = set(new_modules.keys())
+
+    added = sorted(new_keys - old_keys)
+    removed = sorted(old_keys - new_keys)
+    common = old_keys & new_keys
+
+    changed = []
+    for name in sorted(common):
+        old_info = get_module_info(old_modules[name])
+        new_info = get_module_info(new_modules[name])
+
+        changes = {}
+        for key in ("status", "command", "artifact"):
+            if old_info[key] != new_info[key]:
+                changes[key] = {"old": old_info[key], "new": new_info[key]}
+
+        # Duration delta
+        old_dur = old_info["duration"]
+        new_dur = new_info["duration"]
+        if old_dur != new_dur:
+            try:
+                delta = float(new_dur) - float(old_dur) if old_dur is not None and new_dur is not None else None
+            except (ValueError, TypeError):
+                delta = None
+            changes["duration"] = {
+                "old": old_dur,
+                "new": new_dur,
+                "delta": delta,
+            }
+
+        if changes:
+            changed.append({"name": name, "changes": changes})
+
+    return {
+        "added": added,
+        "removed": removed,
+        "changed": changed,
+    }
+
+
+def print_human_diff(diff: dict[str, Any], old_meta: dict[str, Any], new_meta: dict[str, Any]) -> None:
+    """Print a human-readable diff."""
+    if not diff["added"] and not diff["removed"] and not diff["changed"]:
+        print("No differences found.")
+        return
+
+    if diff["added"]:
+        print("Added modules:")
+        for name in diff["added"]:
+            print(f"  + {name}")
+        print()
+
+    if diff["removed"]:
+        print("Removed modules:")
+        for name in diff["removed"]:
+            print(f"  - {name}")
+        print()
+
+    if diff["changed"]:
+        print("Changed modules:")
+        for item in diff["changed"]:
+            name = item["name"]
+            changes = item["changes"]
+            print(f"  {name}:")
+            for key, vals in changes.items():
+                if key == "duration":
+                    old_val = format_duration(vals["old"])
+                    new_val = format_duration(vals["new"])
+                    delta_str = ""
+                    if vals["delta"] is not None:
+                        sign = "+" if vals["delta"] >= 0 else ""
+                        delta_str = f" ({sign}{vals['delta']:.3f}s)"
+                    print(f"    duration: {old_val} -> {new_val}{delta_str}")
+                else:
+                    old_val = vals["old"] if vals["old"] is not None else "N/A"
+                    new_val = vals["new"] if vals["new"] is not None else "N/A"
+                    print(f"    {key}: {old_val} -> {new_val}")
+        print()
+
+
+def main() -> int:
+    parser = argparse.ArgumentParser(
+        description="Compare two diagnostic metadata JSON files.",
+    )
+    parser.add_argument("old_json", help="Path to the old diagnostic metadata JSON file")
+    parser.add_argument("new_json", help="Path to the new diagnostic metadata JSON file")
+    parser.add_argument("--json", action="store_true", help="Output diff as JSON")
+    args = parser.parse_args()
+
+    old_path = Path(args.old_json)
+    new_path = Path(args.new_json)
+
+    # Validate inputs
+    for path in (old_path, new_path):
+        if not path.exists():
+            print(f"Error: File not found: {path}", file=sys.stderr)
+            return 1
+        if not path.is_file():
+            print(f"Error: Not a file: {path}", file=sys.stderr)
+            return 1
+
+    try:
+        old_meta = load_json(old_path)
+    except (json.JSONDecodeError, OSError) as e:
+        print(f"Error: Invalid JSON in {old_path}: {e}", file=sys.stderr)
+        return 1
+
+    try:
+        new_meta = load_json(new_path)
+    except (json.JSONDecodeError, OSError) as e:
+        print(f"Error: Invalid JSON in {new_path}: {e}", file=sys.stderr)
+        return 1
+
+    diff = compute_diff(old_meta, new_meta)
+
+    if args.json:
+        output = {
+            "added": diff["added"],
+            "removed": diff["removed"],
+            "changed": diff["changed"],
+        }
+        print(json.dumps(output, indent=