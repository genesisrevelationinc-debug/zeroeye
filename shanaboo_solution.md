 ```diff
--- a/tools/data_generator.py
+++ b/tools/data_generator.py
@@ -1,4 +1,5 @@
 #!/usr/bin/env python3
+# -*- coding: utf-8 -*-
 """
 Legacy test data generator for development and testing environments.
 Generates realistic-looking market data, orders, trades, and user data
@@ -10,6 +11,9 @@
 WARNING: The generated data is NOT suitable for production use. It does
 NOT follow real market distributions, correlation patterns, or regulatory
 requirements. Using this data for performance testing will produce
 misleading results because the data distribution is uniform rather than
 following the power-law distributions seen in real markets.
+
+Deterministic output:
+    Use --seed <int> to reproduce the exact same dataset across runs.
 """
 
 import argparse
@@ -17,6 +21,7 @@
 import json
 import math
 import os
+import hashlib
 import random
 import sys
 import time
@@ -24,6 +29,11 @@
 from datetime import datetime, timedelta, timezone
 from typing import Any, Dict, List, Optional, Tuple
 
+# ---------------------------------------------------------------------------
+# VERSION
+# ---------------------------------------------------------------------------
+__version__ = "2.0.0"
+
 # ---------------------------------------------------------------------------
 # CONSTANTS
 # ---------------------------------------------------------------------------
@@ -86,6 +96,9 @@
            "fictitious.co", "imaginary.app", "pretend.tech", "dummy.biz",
            "simulated.com", "testmail.com", "inbox.test"]
 
+# Seed used for the current run (global so metadata can reference it)
+_CURRENT_SEED: Optional[int] = None
+
 
 # ---------------------------------------------------------------------------
 # UTILITIES
@@ -93,6 +106,7 @@
 
 def parse_args() -> argparse.Namespace:
     parser = argparse.ArgumentParser(description="Generate synthetic market data")
+    parser.add_argument("--seed", type=int, default=None, help="Random seed for reproducible output")
     parser.add_argument("--output-dir", type=str, default="data/test", help="Output directory")
     parser.add_argument("--num-orders", type=int, default=1000, help="Number of orders")
     parser.add_argument("--num-trades", type=int, default=5000, help="Number of trades")
@@ -101,6 +115,7 @@ def parse_args() -> argparse.Namespace:
     parser.add_argument("--start-date", type=str, default="2023-01-01", help="Start date")
     parser.add_argument("--end-date", type=str, default="2023-12-31", help="End date")
     parser.add_argument("--format", type=str, choices=["json", "csv", "both"], default="both")
+    parser.add_argument("--print-seed", action="store_true", help="Print the seed used and exit (or print auto-generated seed when no --seed given)")
     return parser.parse_args()
 
 
@@ -108,6 +123,7 @@ def parse_args() -> argparse.Namespace:
 # DATA GENERATION
 # ---------------------------------------------------------------------------
 
+
 def generate_timestamp(start: datetime, end: datetime, rng: random.Random) -> datetime:
     """Generate a random timestamp between start and end."""
     delta = end - start
@@ -115,7 +131,7 @@ def generate_timestamp(start: datetime, end: datetime, rng: random.Random) ->
     return start + timedelta(seconds=seconds)
 
 
-def generate_order(instruments: List[Dict], rng: random.Random, start: datetime, end: datetime) -> Dict[str, Any]:
+def generate_order(instruments: List[Dict], rng: random.Random, start: datetime, end: datetime, seed: Optional[int] = None) -> Dict[str, Any]:
     """Generate a single synthetic order."""
     instrument = rng.choice(instruments)
     side = rng.choice(ORDER_SIDES)
@@ -123,6 +139,7 @@ def generate_order(instruments: List[Dict], rng: random.Random, start: datetime,
     status = rng.choice(ORDER_STATUSES)
     tif = rng.choice(TIME_IN_FORCE)
     ts = generate_timestamp(start, end, rng)
+    # Deterministic: use rng for all random choices
     price = round(instrument["price"] * (1 + rng.uniform(-0.1, 0.1)), 4)
     quantity = round(rng.uniform(1, 100) * instrument["lot_size"], 4)
     
@@ -139,6 +156,8 @@ def generate_order(instruments: List[Dict], rng: random.Random, start: datetime,
         "created_at": ts.isoformat(),
         "updated_at": (ts + timedelta(seconds=rng.randint(0, 3600))).isoformat(),
     }
+    if seed is not None:
+        order["_seed"] = seed
     return order
 
 
@@ -196,6 +215,8 @@ def generate_user(rng: random.Random) -> Dict[str, Any]:
         "created_at": created_at.isoformat(),
         "updated_at": updated_at.isoformat(),
     }
+    if _CURRENT_SEED is not None:
+        user["_seed"] = _CURRENT_SEED
     return user
 
 
@@ -204,6 +225,7 @@ def generate_user(rng: random.Random) -> Dict[str, Any]:
 # ---------------------------------------------------------------------------
 
 def write_json(data: Dict[str, Any], path: str) -> None:
+    """Write data to a JSON file with deterministic key ordering."""
     with open(path, "w") as f:
         json.dump(data, f, indent=2, sort_keys=True)
         f.write("\n")
@@ -224,6 +246,7 @@ def write_csv(data: List[Dict[str, Any]], path: str) -> None:
 # ---------------------------------------------------------------------------
 
 def generate_all(args: argparse.Namespace) -> Dict[str, Any]:
+    global _CURRENT_SEED
     output_dir = args.output_dir
     os.makedirs(output_dir, exist_ok=True)
 
@@ -231,8 +254,22 @@ def generate_all(args: argparse.Namespace) -> Dict[str, Any]:
     end = datetime.strptime(args.end_date, "%Y-%m-%d").replace(tzinfo=timezone.utc)
     if end <= start:
         raise ValueError("end_date must be after start_date")
+    
+    # Handle seed
+    if args.seed is not None:
+        seed = args.seed
+    else:
+        # Generate a deterministic seed from current time if not provided
+        seed = int(time.time() * 1000) % (2**31)
+    
+    _CURRENT_SEED = seed
+    
+    if args.print_seed:
+        print(f"Seed: {seed}")
+        if args.seed is None:
+            print("Re-run with: --seed", seed)
+        return {}
 
+    # Create RNG with seed
     rng = random.Random(seed)
 
     # Generate data
@@ -240,7 +277,7 @@ def generate_all(args: argparse.Namespace) -> Dict[str, Any]:
     trades = [generate_trade(orders, rng, start, end) for