```diff
--- /dev/null
+++ b/tools/diagnostic_diff.py
@@ -0,0 +1,237 @@
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
+def load_json(path: str) -> dict[str, Any]:
+    """Load and parse a JSON file, exiting with error if missing or invalid."""
+    p = Path(path)
+    if not p.exists():
+        print(f"Error: file not found: {path}", file=sys.stderr)
+        sys.exit(1)
+    try:
+        with p.open("r", encoding="utf-8") as fh:
+            return json.load(fh)
+    except json.JSONDecodeError as exc:
+        print(f"Error: invalid JSON in {path}: {exc}", file=sys.stderr)
+        sys.exit(1)
+
+
+def _module_key(module: dict) -> str:
+    """Return a stable sort key for a module entry."""
+    return module.get("name", module.get("module", ""))
+
+
+def _index_modules(modules: list[dict]) -> dict[str, dict]:
+    """Build a name→module dict from a list of module objects."""
+    indexed: dict[str, dict] = {}
+    for mod in modules:
+        key = _module_key(mod)
+        if key:
+            indexed[key] = mod
+    return indexed
+
+
+def _safe_get(mod: dict, *keys: str, default: Any = None) -> Any:
+    """Safely traverse nested keys in a dict."""
+    for key in keys:
+        if isinstance(mod, dict):
+            mod = mod.get(key, default)
+        else:
+            return default
+    return mod
+
+
+def _fmt_duration(seconds: float | None) -> str:
+    """Format a duration value for display."""
+    if seconds is None:
+        return "N/A"
+    return f"{seconds:.3f}s"
+
+
+def _delta_str(old_val: Any, new_val: Any) -> str:
+    """Return a human-readable delta string."""
+    try:
+        old_f = float(old_val)
+        new_f = float(new_val)
+        delta = new_f - old_f
+        sign = "+" if delta >= 0 else ""
+        return f"{sign}{delta:.3f}s"
+    except (TypeError, ValueError):
+        return f"{old_val} → {new_val}"
+
+
+def diff_modules(
+    old_modules: list[dict],
+    new_modules: list[dict],
+) -> dict[str, Any]:
+    """Compare two module lists and return a structured diff."""
+    old_idx = _index_modules(old_modules)
+    new_idx = _index_modules(new_modules)
+
+    old_names = set(old_idx.keys())
+    new_names = set(new_idx.keys())
+
+    added = sorted(new_names - old_names)
+    removed = sorted(old_names - new_names)
+    common = sorted(old_names & new_names)
+
+    changed: list[dict] = []
+    for name in common:
+        old_mod = old_idx[name]
+        new_mod = new_idx[name]
+        changes: dict[str, Any] = {}
+
+        # Status
+        old_status = _safe_get(old_mod, "status")
+        new_status = _safe_get(new_mod, "status")
+        if old_status != new_status:
+            changes["status"] = {"old": old_status, "new": new_status}
+
+        # Duration
+        old_dur = _safe_get(old_mod, "duration_seconds", "duration")
+        new_dur = _safe_get(new_mod, "duration_seconds", "duration")
+        if old_dur != new_dur:
+            changes["duration"] = {"old": old_dur, "new": new_dur}
+
+        # Commands
+        old_cmd = _safe_get(old_mod, "command", "build_cmd")
+        new_cmd = _safe_get(new_mod, "command", "build_cmd")
+        if old_cmd != new_cmd:
+            changes["command"] = {"old": old_cmd, "new": new_cmd}
+
+        # Artifact names
+        old_art = _safe_get(old_mod, "artifact", "artifacts")
+        new_art = _safe_get(new_mod, "artifact", "artifacts")
+        if old_art != new_art:
+            changes["artifact"] = {"old": old_art, "new": new_art}
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
+def print_human_diff(diff: dict[str, Any]) -> None:
+    """Print a human-readable diff to stdout."""
+    added = diff["added"]
+    removed = diff["removed"]
+    changed = diff["changed"]
+
+    if not added and not removed and not changed:
+        print("No differences found between the two diagnostic files.")
+        return
+
+    if added:
+        print("=== Added Modules ===")
+        for name in added:
+            print(f"  + {name}")
+        print()
+
+    if removed:
+        print("=== Removed Modules ===")
+        for name in removed:
+            print(f"  - {name}")
+        print()
+
+    if changed:
+        print("=== Changed Modules ===")
+        for entry in changed:
+            name = entry["name"]
+            changes = entry["changes"]
+            print(f"  * {name}")
+            if "status" in changes:
+                print(f"      status:  {changes['status']['old']} → {changes['status']['new']}")
+            if "duration" in changes:
+                old_d = changes["duration"]["old"]
+                new_d = changes["duration"]["new"]
+                print(f"      duration: {_fmt_duration(old_d)} → {_fmt_duration(new_d)}  (Δ {_delta_str(old_d, new_d)})")
+            if "command" in changes:
+                print(f"      command:  {changes['command']['old']} → {changes['command']['new']}")
+            if "artifact" in changes:
+                print(f"      artifact: {changes['artifact']['old']} → {changes['artifact']['new']}")
+            print()
+
+
+def main()