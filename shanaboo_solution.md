 ```diff
--- a/tools/data_generator.py
+++ b/tools/data_generator.py
@@ -1,4 +1,5 @@
 #!/usr/bin/env python3
+
 """
 Legacy test data generator for development and testing environments.
 Generates realistic-looking market data, orders, trades, and user data
@@ -7,6 +8,8 @@
 
 The data generator uses seeded random number generation to produce
 deterministic output for reproducible test scenarios. Change the seed
 to generate different datasets.
 
+When no seed is provided, a random seed is generated and can be printed
+using --print-seed for later reproduction.
+
 WARNING: The generated data is NOT suitable for production use. It does
 NOT follow real market distributions, correlation patterns, or regulatory
 requirements. Using this data for performance testing will produce
@@ -14,7 +17,7 @@
 """
 
 import argparse
 import csv
 import json
 import math
 import os
 import random
 import sys
 import time
 from datetime import datetime, timedelta, timezone
 from typing import Any, Dict, List, Optional, Tuple
 
 # ---------------------------------------------------------------------------
 # CONSTANTS
 # ---------------------------------------------------------------------------
 
 INSTRUMENTS = [
@@ -96,7 +99,7 @@
           "Wright", "Scott", "Torres", "Hill", "Green", "Adams", "Baker", "Nelson",
           "Carter", "Mitchell", "Roberts", "Turner", "Phillips", "Campbell"]
 
 DOMAINS = ["example.com", "test.org", "demo.net", "sample.io", "mock.dev",
            "fictitious.co", "imaginary.app", "pretend.tech", "dummy.biz",
            "simulated.com", "testmail.com", "inbox.test"]
 
 # ---------------------------------------------------------------------------
 # HELPERS
 # ---------------------------------------------------------------------------
 
 def _round_to_tick(value: float, tick_size: float) -> float:
     """Round a value to the nearest tick size."""
     return round(value / tick_size) * tick_size
 
 
 def _generate_timestamp(start: datetime, end: datetime, rng: random.Random) -> str:
     """Generate a random ISO timestamp between start and end."""
     delta = (end - start).total_seconds()
     offset = rng.uniform(0, delta)
     ts = start + timedelta(seconds=offset)
     return ts.isoformat()
 
 
 # ---------------------------------------------------------------------------
 # GENERATORS
 # ---------------------------------------------------------------------------
 
 class DataGenerator:
     """Deterministic data generator using a seeded random number generator."""
 
     def __init__(self, seed: int):
         self.seed = seed
         self.rng = random.Random(seed)
 
     def reset(self) -> None:
         """Reset the RNG to its initial state for reproducibility."""
         self.rng = random.Random(self.seed)
 
     def generate_orders(self, count: int) -> List[Dict[str, Any]]:
         """Generate a list of order dictionaries."""
         self.reset()
         orders = []
         now = datetime.now(timezone.utc)
         start = now - timedelta(days=30)
         for i in range(count):
             instrument = self.rng.choice(INSTRUMENTS)
             side = self.rng.choice(ORDER_SIDES)
             order_type = self.rng.choice(ORDER_TYPES)
             status = self.rng.choice(ORDER_STATUSES)
             tif = self.rng.choice(TIME_IN_FORCE)
             price = _round_to_tick(
                 self.rng.gauss(instrument["price"], instrument["price"] * instrument["vol"] * 0.01),
                 instrument["tick_size"],
             )
             price = max(price, instrument["tick_size"])
             quantity = _round_to_tick(
                 self.rng.lognormvariate(0, 1) * instrument["lot_size"] * 100,
                 instrument["lot_size"],
             )
             quantity = max(quantity, instrument["lot_size"])
             filled_qty = 0.0
             if status == "filled":
                 filled_qty = quantity
             elif status == "partially_filled":
                 filled_qty = _round_to_tick(quantity * self.rng.random(), instrument["lot_size"])
             order = {
                 "order_id": f"ORD-{i+1:08d}",
                 "symbol": instrument["symbol"],
                 "side": side,
                 "type": order_type,
                 "status": status,
                 "price": round(price, 8),
                 "quantity": round(quantity, 8),
                 "filled_quantity": round(filled_qty, 8),
                 "time_in_force": tif,
                 "created_at": _generate_timestamp(start, now, self.rng),
                 "updated_at": _generate_timestamp(start, now, self.rng),
             }
             orders.append(order)
         return orders
 
     def generate_trades(self, count: int) -> List[Dict[str, Any]]:
         """Generate a list of trade dictionaries."""
         self.reset()
         trades = []
         now = datetime.now(timezone.utc)
         start = now - timedelta(days=30)
         for i in range(count):
             instrument = self.rng.choice(INSTRUMENTS)
             side = self.rng.choice(ORDER_SIDES)
             price = _round_to_tick(
                 self.rng.gauss(instrument["price"], instrument["price"] * instrument["vol"] * 0.01),
                 instrument["tick_size"],
             )
             price = max(price, instrument["tick_size"])
             quantity = _round_to_tick(
                 self.rng.lognormvariate(0, 1) * instrument["lot_size"] * 100,
                 instrument["lot_size"],
             )
             quantity = max(quantity, instrument["lot_size"])
             trade = {
                 "trade_id": f"TRD-{i+1:08d}",
                 "symbol": instrument["symbol"],
                 "side": side,
                 "price": round(price, 8),
                 "quantity": round(quantity, 8),
                 "timestamp": _generate_timestamp(start, now, self.rng),
             }
             trades.append(trade)
         return trades
 
     def generate_users(self, count: int) -> List[Dict[str, Any]]:
         """Generate a list of user dictionaries."""
         self.reset()
         users = []
         for i in range(count):
             first = self.rng.choice(FIRST_NAMES)
             last = self.rng.choice(LAST_NAMES)
             domain = self.rng.choice(DOMAINS)
             user = {
                 "user_id": f"USR-{i+1:08d}",
                 "name": f"{first} {last}",
                 "email": f"{first.lower()}.{last.lower()}@{domain}",
                 "role": self.rng.choice(["admin", "trader", "viewer", "compliance"]),
                 "created_at": (datetime.now(timezone.utc) - timedelta(days=self.rng.randint(1, 365))).isoformat(),
                 "active": self.rng.random() > 0.1,
             }
             users.append(user)
         return users
 
     def generate_market_data(self, count: int) -> List[Dict[str, Any]]:
         """Generate a list of market data