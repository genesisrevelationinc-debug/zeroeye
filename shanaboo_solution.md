 ```diff
--- a/tools/data_generator.py
+++ b/tools/data_generator.py
@@ -1,4 +1,5 @@
 #!/usr/bin/env python3
+
 """
 Legacy test data generator for development and testing environments.
 Generates realistic-looking market data, orders, trades, and user data
@@ -8,7 +9,8 @@
 The data generator uses seeded random number generation to produce
 deterministic output for reproducible test scenarios. Change the seed
 to generate different datasets.
-
+When a seed is provided via --seed, output is byte-for-byte reproducible.
+Use --print-seed to discover the seed used for a random run.
 WARNING: The generated data is NOT suitable for production use. It does
 NOT follow real market distributions, correlation patterns, or regulatory
 requirements. Using this data for performance testing will produce
@@ -16,6 +18,7 @@
 following the power-law distributions seen in real markets.
 """
 
+
 import argparse
 import csv
 import json
@@ -25,6 +28,7 @@
 import sys
 import time
 from datetime import datetime, timedelta, timezone
+from hashlib import sha256
 from typing import Any, Dict, List, Optional, Tuple
 
 # ---------------------------------------------------------------------------
@@ -85,6 +89,7 @@
 # ---------------------------------------------------------------------------
 
 def parse_args() -> argparse.Namespace:
+    """Parse and return command line arguments."""
     parser = argparse.ArgumentParser(
         description="Generate synthetic market data for testing and development."
     )
@@ -96,6 +101,20 @@ def parse_args() -> argparse.Namespace:
     parser.add_argument("--output-dir", default="data/test", help="Output directory")
     parser.add_argument("--format", choices=["json", "csv"], default="json", help="Output format")
     parser.add_argument("--compress", action="store_true", help="Gzip output files")
+    parser.add_argument(
+        "--seed",
+        type=int,
+        default=None,
+        help="Random seed for deterministic output. If omitted, a random seed is chosen.",
+    )
+    parser.add_argument(
+        "--print-seed",
+        action="store_true",
+        help="Print the seed used (random or provided) to stderr before generating data.",
+    )
+    return parser.parse_args()
+
+
 # ---------------------------------------------------------------------------
 # UTILITIES
 # ---------------------------------------------------------------------------
@@ -103,6 +122,7 @@
 def generate_id(prefix: str = "id") -> str:
     """Generate a unique identifier with an optional prefix."""
     return f"{prefix}_{random.randint(100000, 999999)}"
+
 
 def generate_timestamp(start: datetime, end: datetime) -> datetime:
     """Generate a random timestamp between start and end."""
@@ -110,6 +130,7 @@ def generate_timestamp(start: datetime, end: datetime) -> datetime:
     delta_seconds = (end - start).total_seconds()
     return start + timedelta(seconds=random.random() * delta_seconds)
 
+
 def round_to_tick(price: float, tick_size: float) -> float:
     """Round a price to the nearest tick size."""
     return round(price / tick_size) * tick_size
@@ -118,6 +139,7 @@ def round_to_tick(price: float, tick_size: float) -> float:
 # DATA GENERATORS
 # ---------------------------------------------------------------------------
 
+
 def generate_instruments(count: int = 10) -> List[Dict[str, Any]]:
     """Generate synthetic instrument definitions."""
     instruments = []
@@ -138,6 +160,7 @@ def generate_instruments(count: int = 10) -> List[Dict[str, Any]]:
         instruments.append(inst)
     return instruments
 
+
 def generate_users(count: int = 100) -> List[Dict[str, Any]]:
     """Generate synthetic user data."""
     users = []
@@ -157,6 +180,7 @@ def generate_users(count: int = 100) -> List[Dict[str, Any]]:
     })
     return users
 
+
 def generate_orders(instruments: List[Dict[str, Any]], count: int = 1000) -> List[Dict[str, Any]]:
     """Generate synthetic order data."""
     orders = []
@@ -183,6 +207,7 @@ def generate_orders(instruments: List[Dict[str, Any]], count: int = 1000) -> Lis
         orders.append(order)
     return orders
 
+
 def generate_trades(orders: List[Dict[str, Any]], count: int = 500) -> List[Dict[str, Any]]:
     """Generate synthetic trade data based on existing orders."""
     trades = []
@@ -210,6 +235,7 @@ def generate_trades(orders: List[Dict[str, Any]], count: int = 500) -> List[Dict
         trades.append(trade)
     return trades
 
+
 def generate_market_data(instruments: List[Dict[str, Any]], points: int = 1000) -> List[Dict[str, Any]]:
     """Generate synthetic market data (OHLC) for instruments."""
     market_data = []
@@ -240,6 +266,7 @@ def generate_market_data(instruments: List[Dict[str, Any]], points: int = 1000) -
             market_data.append(point)
     return market_data
 
+
 def generate_positions(users: List[Dict[str, Any]], instruments: List[Dict[str, Any]], count: int = 200) -> List[Dict[str, Any]]:
     """Generate synthetic position data."""
     positions = []
@@ -264,6 +291,7 @@ def generate_positions(users: List[Dict[str, Any]], instruments: List[Dict[str,
         positions.append(position)
     return positions
 
+
 def generate_risk_metrics(users: List[Dict[str, Any]], positions: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
     """Generate synthetic risk metrics for users positions."""
     risk_metrics = []
@@ -289,6 +317,7 @@ def generate_risk_metrics(users: List[Dict[str, Any]], positions: List[Dict[str,
         risk_metrics.append(metric)
     return risk_metrics
 
+
 # ---------------------------------------------------------------------------
 # OUTPUT FORMATTERS
 # ---------------------------------------------------------------------------
@@ -296,6 +325,7 @@ def generate_risk_metrics(users: List[Dict[str, Any]], positions: List[Dict[str,
 def write_json(data: Dict[str, Any], path: str) -> None:
     """Write data to a JSON file with deterministic key ordering."""
     with open(path, "w", encoding="utf-8") as f:
+        # Seed is embedded in metadata when available
         json.dump(data, f, indent=2, sort_keys=True, ensure_ascii=False)
         f.write("\n")
 
@@ -303,6 +333,7 @@ def write_csv(data: List[Dict[str, Any]], path: str) -> None:
     """Write data to a CSV file."""
     if not data:
         return
+
     keys = list(data[0].keys())
     with open(path, "w", newline="", encoding="utf-8") as f:
         writer = csv.DictWriter(f, fieldnames=keys)
@@ -310,6 +341