# DP-700: Exercise Solutions & Reference Implementations
> Complete working solutions for all exercises  
> Python 3.10+ | PySpark 3.5+ | Delta Lake 3.x | DuckDB 0.10+

---

## Setup: Install All Dependencies

```bash
pip install pyspark delta-spark duckdb pandas faker requests matplotlib \
            statsmodels pyarrow fastparquet python-dateutil
```

---

# MODULE 1 SOLUTIONS

---

## Solution 1.1 — OneLake Architecture Setup

```python
# solution_1_1_onelake_setup.py
import os, requests, json
from pyspark.sql import SparkSession
from pyspark.sql.functions import col
from delta import configure_spark_with_delta_pip, DeltaTable

# ── Spark Session ──────────────────────────────────────────────────────────────
builder = (
    SparkSession.builder
    .appName("DP700_Exercise_1_1")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
)
spark = configure_spark_with_delta_pip(builder).getOrCreate()
spark.sparkContext.setLogLevel("ERROR")

BASE_PATH = "/tmp/onelake_sim"
BRONZE = f"{BASE_PATH}/Bronze_LH/Tables"
SILVER = f"{BASE_PATH}/Silver_LH/Tables"

# ── Task 1: Download NYC Taxi Parquet ─────────────────────────────────────────
url = "https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2023-01.parquet"
local_path = "/tmp/yellow_tripdata_2023-01.parquet"

if not os.path.exists(local_path):
    print("Downloading NYC Taxi Jan 2023...")
    with requests.get(url, stream=True) as r:
        r.raise_for_status()
        with open(local_path, "wb") as f:
            for chunk in r.iter_content(chunk_size=8192):
                f.write(chunk)
    print(f"Downloaded to {local_path}")

# ── Task 2: Read Parquet, Print Schema and Row Count ─────────────────────────
df_raw = spark.read.parquet(local_path)
print("\n=== RAW SCHEMA ===")
df_raw.printSchema()
print(f"Row count: {df_raw.count():,}")

# ── Task 3: Write to Bronze Delta Table ───────────────────────────────────────
bronze_path = f"{BRONZE}/yellow_taxi_raw"
(df_raw.write
    .format("delta")
    .mode("overwrite")
    .save(bronze_path))
print(f"\nBronze table written to: {bronze_path}")

# ── Task 4: Inspect Delta Transaction Log ────────────────────────────────────
log_path = f"{bronze_path}/_delta_log/00000000000000000000.json"
with open(log_path) as f:
    for line in f:
        entry = json.loads(line)
        print(json.dumps(entry, indent=2))
# The log contains:
#   {"commitInfo": {...}}  — who wrote, when, what operation
#   {"metaData": {...}}    — schema (as JSON), partitionColumns, table format config
#   {"add": {...}}         — one entry per Parquet file added (stats, path, size)

# ── Task 5: Silver Transformation ────────────────────────────────────────────
REQUIRED_COLS = [
    "tpep_pickup_datetime", "tpep_dropoff_datetime",
    "passenger_count", "trip_distance", "fare_amount",
    "total_amount", "PULocationID", "DOLocationID"
]

df_bronze = spark.read.format("delta").load(bronze_path)
missing = [c for c in REQUIRED_COLS if c not in df_bronze.columns]
assert not missing, f"Missing columns: {missing}"

df_silver = (df_bronze
    .select(REQUIRED_COLS)
    .dropna(subset=["tpep_pickup_datetime"])
    .filter(col("trip_distance") > 0)
    .filter(col("fare_amount") > 0))

silver_path = f"{SILVER}/yellow_taxi_cleaned"
(df_silver.write
    .format("delta")
    .mode("overwrite")
    .save(silver_path))
print(f"\nSilver rows: {df_silver.count():,}")

# ── Task 6: Delta History ─────────────────────────────────────────────────────
DeltaTable.forPath(spark, silver_path).history().show(truncate=False)

"""
TASK 7 ANSWER — SQL Analytics Endpoint:
The SQL Analytics Endpoint is an auto-provisioned, read-only T-SQL interface
Fabric generates automatically for every Lakehouse. It reads Delta files directly
from OneLake and exposes them as SQL tables without any data movement.

Why you can't INSERT:
- Write operations must go through the Spark runtime (Notebooks) or Delta Lake API
  to maintain transaction log integrity and ACID guarantees.
- Allowing raw SQL INSERTs would bypass the Delta transaction log, risking corruption.
- Use Fabric Warehouse (full DML) if you need INSERT/UPDATE/DELETE via T-SQL.
"""
```

---

## Solution 1.2 — Capacity Smoothing Simulator

```python
# solution_1_2_capacity_simulator.py
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import List, Literal
import random
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches

@dataclass
class CapacityEvent:
    timestamp: datetime
    operation: str
    cu_seconds: float
    status: Literal["OK", "THROTTLED"] = "OK"

SKU_CAPACITY = {"F4": 4, "F8": 8, "F16": 16, "F64": 64}

def generate_events(n: int = 500) -> List[CapacityEvent]:
    base = datetime(2024, 1, 15, 0, 0, 0)
    events = []
    for i in range(n):
        offset_minutes = random.uniform(0, 1440)
        ts = base + timedelta(minutes=offset_minutes)
        hour = ts.hour
        if 8 <= hour <= 11:
            cu = random.uniform(20, 60)
            op = random.choice(["ETL_Pipeline", "Dataflow_Gen2", "Notebook_Silver"])
        elif 12 <= hour <= 14:
            cu = random.uniform(5, 15)
            op = random.choice(["Power_BI_Refresh", "SQL_Query"])
        else:
            cu = random.uniform(0.5, 5)
            op = random.choice(["Scheduled_Refresh", "API_Call"])
        events.append(CapacityEvent(timestamp=ts, operation=op, cu_seconds=cu))
    return sorted(events, key=lambda e: e.timestamp)


class SmoothingEngine:
    def __init__(self, capacity_sku: str = "F4", smoothing_window_minutes: int = 10):
        self.capacity_cu = SKU_CAPACITY[capacity_sku]
        self.sku = capacity_sku
        self.window_sec = smoothing_window_minutes * 60
        self.max_debt = self.capacity_cu * 10 * 60
        self.debt = 0.0
        self.processed: List[CapacityEvent] = []

    def process(self, events: List[CapacityEvent]) -> List[CapacityEvent]:
        self.processed = []
        for i, event in enumerate(events):
            window_start = event.timestamp - timedelta(seconds=self.window_sec)
            window_events = [e for e in events[:i] if e.timestamp >= window_start]
            smoothed_cu = sum(e.cu_seconds for e in window_events) / self.window_sec

            excess = max(0, smoothed_cu - self.capacity_cu)
            self.debt += excess * 60
            self.debt = max(0, self.debt - self.capacity_cu * 0.5)

            if self.debt > self.max_debt:
                event.status = "THROTTLED"
            self.processed.append(event)
        return self.processed

    def generate_report(self) -> dict:
        throttled = [e for e in self.processed if e.status == "THROTTLED"]
        total_cu = sum(e.cu_seconds for e in self.processed)
        peak = max((e.cu_seconds for e in self.processed), default=0)
        return {
            "sku": self.sku,
            "capacity_cu": self.capacity_cu,
            "total_events": len(self.processed),
            "throttled_events": len(throttled),
            "throttle_rate_pct": round(len(throttled) / len(self.processed) * 100, 2),
            "total_cu_consumed": round(total_cu, 2),
            "peak_event_cu": round(peak, 2),
        }


if __name__ == "__main__":
    events = generate_events(500)
    for sku in ["F4", "F8", "F16"]:
        engine = SmoothingEngine(capacity_sku=sku)
        processed = engine.process(list(events))  # pass copy so debt resets
        report = engine.generate_report()
        print(f"\n{'='*50}\nSKU: {sku}")
        for k, v in report.items():
            print(f"  {k}: {v}")
```

---

## Solution 2.1 — Row-Level Security with DuckDB

```python
# solution_2_1_rls.py
import duckdb
import pandas as pd
import requests
import time

url = "https://raw.githubusercontent.com/microsoft/sql-server-samples/master/samples/databases/wide-world-importers/csv-files/orders.csv"
df = pd.read_csv(url)

con = duckdb.connect(":memory:")
con.execute("CREATE TABLE orders AS SELECT * FROM df")
con.execute("""
    ALTER TABLE orders ADD COLUMN region VARCHAR;
    UPDATE orders SET region = CASE (rowid % 3)
        WHEN 0 THEN 'APAC'
        WHEN 1 THEN 'EMEA'
        ELSE 'AMER' END;
""")
con.execute("""
    CREATE TABLE rls_users (username VARCHAR PRIMARY KEY, region VARCHAR);
    INSERT INTO rls_users VALUES
        ('user_apac', 'APAC'),
        ('user_emea', 'EMEA'),
        ('user_amer', 'AMER'),
        ('admin', NULL);
""")

def query_as_user(username: str) -> pd.DataFrame:
    return con.execute(f"""
        SELECT o.*
        FROM orders o
        WHERE EXISTS (
            SELECT 1 FROM rls_users u
            WHERE u.username = '{username}'
              AND (u.region IS NULL OR u.region = o.region)
        )
    """).df()

for user in ["user_apac", "user_emea", "admin"]:
    t0 = time.perf_counter()
    result = query_as_user(user)
    elapsed = time.perf_counter() - t0
    regions = result["region"].unique().tolist() if "region" in result.columns else []
    print(f"User: {user:12s} | Rows: {len(result):6,} | Regions: {regions} | {elapsed*1000:.1f}ms")

# Fabric Warehouse equivalent T-SQL
FABRIC_RLS_DDL = """
CREATE SCHEMA security;
GO
CREATE FUNCTION security.fn_rls_orders(@region VARCHAR(50))
RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS access_result
WHERE @region = (SELECT region FROM security.rls_users WHERE username = USER_NAME())
   OR (SELECT region FROM security.rls_users WHERE username = USER_NAME()) IS NULL;
GO
CREATE SECURITY POLICY orders_rls_policy
ADD FILTER PREDICATE security.fn_rls_orders(region)
ON dbo.orders WITH (STATE = ON);
GO
"""
print("\nFabric Warehouse T-SQL DDL:")
print(FABRIC_RLS_DDL)
```

---

## Solution 3.1 — Full vs Incremental Load

```python
# solution_3_1_incremental_load.py
import os, json, requests
from pyspark.sql import SparkSession, DataFrame
from pyspark.sql.functions import col
from delta import configure_spark_with_delta_pip, DeltaTable

builder = (SparkSession.builder.appName("DP700_3_1")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog"))
spark = configure_spark_with_delta_pip(builder).getOrCreate()
spark.sparkContext.setLogLevel("ERROR")

URLS = {
    "jan": "https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2023-01.parquet",
    "feb": "https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2023-02.parquet",
}
LOCAL = {k: f"/tmp/yellow_taxi_{k}.parquet" for k in URLS}

def download(key):
    if not os.path.exists(LOCAL[key]):
        with requests.get(URLS[key], stream=True) as r:
            r.raise_for_status()
            with open(LOCAL[key], "wb") as f:
                for chunk in r.iter_content(8192): f.write(chunk)

for k in URLS: download(k)

TARGET_PATH    = "/tmp/taxi_incremental"
WATERMARK_PATH = "/tmp/taxi_incremental/_metadata/watermark.json"

def full_load(source_df: DataFrame, target_path: str) -> int:
    source_df.write.format("delta").mode("overwrite").save(target_path)
    return source_df.count()

def save_watermark(path: str, ts: str):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w") as f: json.dump({"watermark": ts}, f)

def load_watermark(path: str) -> str:
    if not os.path.exists(path): return "1900-01-01T00:00:00"
    with open(path) as f: return json.load(f)["watermark"]

def incremental_load(source_df: DataFrame, target_path: str,
                     watermark_col: str, watermark_path: str) -> int:
    last_wm = load_watermark(watermark_path)
    print(f"  Last watermark: {last_wm}")
    incremental = source_df.filter(col(watermark_col) > last_wm)
    cnt = incremental.count()
    if cnt > 0:
        incremental.write.format("delta").mode("append").save(target_path)
        new_wm = source_df.agg({watermark_col: "max"}).collect()[0][0]
        save_watermark(watermark_path, str(new_wm))
        print(f"  New watermark: {new_wm}")
    return cnt

df_jan = spark.read.parquet(LOCAL["jan"])
df_feb = spark.read.parquet(LOCAL["feb"])

print("Step 1: Full load (January)")
n = full_load(df_jan, TARGET_PATH)
wm = str(df_jan.agg({"tpep_pickup_datetime": "max"}).collect()[0][0])
save_watermark(WATERMARK_PATH, wm)
print(f"  Written {n:,} rows, watermark={wm}")

print("\nStep 2: Incremental load (February)")
n_inc = incremental_load(df_feb, TARGET_PATH, "tpep_pickup_datetime", WATERMARK_PATH)
print(f"  Appended {n_inc:,} rows")

print("\nDelta History:")
DeltaTable.forPath(spark, TARGET_PATH).history().select(
    "version", "timestamp", "operation", "operationMetrics"
).show(truncate=False)
```

---

## Solution 3.4 — MERGE / Upsert with SCD Type 2

```python
# solution_3_4_merge.py
import duckdb
import pandas as pd
from faker import Faker
from datetime import datetime

fake = Faker()
con = duckdb.connect(":memory:")

customers = [{
    "customer_id": i, "full_name": fake.name(), "email": fake.email(),
    "phone": fake.phone_number(), "is_active": True,
    "updated_at": datetime(2024, 1, 1), "valid_from": datetime(2024, 1, 1),
    "valid_to": None, "is_current": True,
} for i in range(1, 1001)]
df_target = pd.DataFrame(customers)
con.execute("CREATE TABLE dim_customer AS SELECT * FROM df_target")

updates = (
    [{"customer_id": i, "email": fake.email(), "phone": fake.phone_number(),
      "operation": "UPDATE", "full_name": None, "is_active": True} for i in range(1, 201)] +
    [{"customer_id": i, "email": fake.email(), "phone": fake.phone_number(),
      "operation": "INSERT", "full_name": fake.name(), "is_active": True} for i in range(1001, 1101)] +
    [{"customer_id": i, "email": None, "phone": None,
      "operation": "DELETE", "full_name": None, "is_active": False} for i in range(201, 251)]
)
df_source = pd.DataFrame(updates)
con.execute("CREATE TABLE incoming_updates AS SELECT * FROM df_source")

con.execute("""
    MERGE INTO dim_customer AS target
    USING incoming_updates AS source
    ON target.customer_id = source.customer_id
    WHEN MATCHED AND source.operation = 'DELETE' THEN
        UPDATE SET is_active = FALSE, updated_at = NOW()
    WHEN MATCHED AND source.operation = 'UPDATE' THEN
        UPDATE SET
            email      = COALESCE(source.email, target.email),
            phone      = COALESCE(source.phone, target.phone),
            updated_at = NOW()
    WHEN NOT MATCHED AND source.operation = 'INSERT' THEN
        INSERT (customer_id, full_name, email, phone, is_active, updated_at, valid_from, valid_to, is_current)
        VALUES (source.customer_id, source.full_name, source.email, source.phone,
                TRUE, NOW(), NOW(), NULL, TRUE);
""")

final = con.execute("""
    SELECT COUNT(*) AS total, SUM(CASE WHEN is_active THEN 1 ELSE 0 END) AS active
    FROM dim_customer
""").df()
print("After MERGE:"); print(final.to_string(index=False))

# Idempotency test
before = con.execute("SELECT COUNT(*) FROM dim_customer").fetchone()[0]
con.execute("""
    MERGE INTO dim_customer AS target USING incoming_updates AS source
    ON target.customer_id = source.customer_id
    WHEN MATCHED AND source.operation = 'UPDATE' THEN
        UPDATE SET email = COALESCE(source.email, target.email)
    WHEN NOT MATCHED AND source.operation = 'INSERT' THEN
        INSERT (customer_id, full_name, email, phone, is_active, updated_at, valid_from, valid_to, is_current)
        VALUES (source.customer_id, source.full_name, source.email, source.phone, TRUE, NOW(), NOW(), NULL, TRUE);
""")
after = con.execute("SELECT COUNT(*) FROM dim_customer").fetchone()[0]
assert before == after, "Idempotency failed!"
print(f"\nIdempotency check passed: count unchanged at {after}")

# SCD Type 2
con.execute("""
    UPDATE dim_customer
    SET valid_to = NOW(), is_current = FALSE
    WHERE customer_id IN (SELECT customer_id FROM incoming_updates WHERE operation = 'UPDATE')
    AND is_current = TRUE;
""")
con.execute("""
    INSERT INTO dim_customer
        (customer_id, full_name, email, phone, is_active, updated_at, valid_from, valid_to, is_current)
    SELECT s.customer_id, t.full_name,
           COALESCE(s.email, t.email), COALESCE(s.phone, t.phone),
           TRUE, NOW(), NOW(), NULL, TRUE
    FROM incoming_updates s
    JOIN (SELECT * FROM dim_customer WHERE is_current = FALSE) t
      ON s.customer_id = t.customer_id
    WHERE s.operation = 'UPDATE';
""")
print("\nSCD Type 2 — customer_id=1 history:")
print(con.execute("""
    SELECT customer_id, email, valid_from, valid_to, is_current
    FROM dim_customer WHERE customer_id = 1 ORDER BY valid_from
""").df().to_string(index=False))
```

---

## Solution 4.2 — Windowing Functions

```python
# solution_4_2_windowing.py
import random
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import List, Dict
import pandas as pd

@dataclass
class StockTick:
    timestamp: datetime
    price: float
    volume: int

def generate_stream(n: int = 10000) -> List[StockTick]:
    start, price, current_ts = datetime(2024, 1, 15, 9, 30), 100.0, datetime(2024, 1, 15, 9, 30)
    ticks = []
    for _ in range(n):
        price = max(0.01, price + random.gauss(0, 0.05))
        current_ts += timedelta(seconds=random.uniform(0.5, 2.0))
        ticks.append(StockTick(current_ts, round(price, 4), random.randint(100, 1000)))
    return ticks

stream = generate_stream()

# 1. Tumbling Window (5 min OHLCV)
def tumbling_window(ticks: List[StockTick], window_min: int = 5) -> List[Dict]:
    window_sec = window_min * 60
    base = ticks[0].timestamp.replace(second=0, microsecond=0)
    base = base.replace(minute=(base.minute // window_min) * window_min)
    buckets: Dict[datetime, List[StockTick]] = {}
    for tick in ticks:
        elapsed = (tick.timestamp - base).total_seconds()
        bucket_ts = base + timedelta(seconds=int(elapsed // window_sec) * window_sec)
        buckets.setdefault(bucket_ts, []).append(tick)
    return [{"window_start": ts, "window_end": ts + timedelta(minutes=window_min),
             "open": e[0].price, "high": max(t.price for t in e),
             "low": min(t.price for t in e), "close": e[-1].price,
             "volume": sum(t.volume for t in e)}
            for ts, e in sorted(buckets.items())]

# 2. Hopping Window (10 min / 5 min)
def hopping_window(ticks, window_min=10, hop_min=5):
    results, hop, window = [], timedelta(minutes=hop_min), timedelta(minutes=window_min)
    current = ticks[0].timestamp
    while current <= ticks[-1].timestamp:
        w_end = current + window
        events = [t for t in ticks if current <= t.timestamp < w_end]
        if events:
            prices = [e.price for e in events]
            results.append({"window_start": current, "avg": round(sum(prices)/len(prices), 4),
                            "tick_count": len(events)})
        current += hop
    return results

# 3. Sliding Window Alert (10% drop)
def trigger_alert(event: StockTick, drop_pct: float) -> dict:
    return {"alert_type": "PRICE_DROP", "timestamp": event.timestamp.isoformat(),
            "current_price": event.price, "drop_pct": round(drop_pct * 100, 2),
            "severity": "HIGH" if drop_pct > 0.15 else "MEDIUM"}

def sliding_alerts(ticks, window_min=5, threshold=0.10):
    window = timedelta(minutes=window_min)
    alerts = []
    for i, tick in enumerate(ticks):
        w = [t for t in ticks[:i+1] if t.timestamp >= tick.timestamp - window]
        if len(w) < 2: continue
        drop = (w[0].price - tick.price) / w[0].price
        if drop >= threshold:
            alerts.append(trigger_alert(tick, drop))
    return alerts

candles = tumbling_window(stream)
print(f"Tumbling (5min): {len(candles)} candles")

hops = hopping_window(stream)
total_assignments = sum(r["tick_count"] for r in hops)
print(f"Hopping: {len(hops)} windows, {total_assignments} assignments (~2x {len(stream)})")

alerts = sliding_alerts(stream)
print(f"Sliding alerts fired: {len(alerts)}")

KQL = """
// KQL Tumbling 5-minute OHLCV
StockTicks
| summarize high=max(price), low=min(price), volume=sum(volume), ticks=count()
  by bin(timestamp, 5m)
| render timechart
"""
print(KQL)
```

---

## Solution 5.3 — Delta Vacuum and Time Travel

```python
# solution_5_3_vacuum.py
import os, glob, requests
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, lit
from delta import configure_spark_with_delta_pip, DeltaTable

builder = (SparkSession.builder.appName("DP700_5_3")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .config("spark.databricks.delta.retentionDurationCheck.enabled", "false"))
spark = configure_spark_with_delta_pip(builder).getOrCreate()
spark.sparkContext.setLogLevel("ERROR")

path_jan = "/tmp/taxi_jan.parquet"
if not os.path.exists(path_jan):
    with requests.get("https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2023-01.parquet", stream=True) as r:
        with open(path_jan, "wb") as f:
            for chunk in r.iter_content(8192): f.write(chunk)

TABLE = "/tmp/delta_lifecycle"
df_jan = spark.read.parquet(path_jan)

# V0: Initial write
df_jan.write.format("delta").mode("overwrite").save(TABLE)
print(f"V0: {df_jan.count():,} rows")

# V1: Append (simulate Feb = small subset)
df_jan.limit(50000).write.format("delta").mode("append").save(TABLE)
print("V1: Append 50K rows")

# V2: MERGE update
dt = DeltaTable.forPath(spark, TABLE)
updates = spark.read.format("delta").load(TABLE).limit(1000).withColumn("total_amount", col("total_amount") * 1.05)
(dt.alias("t").merge(updates.alias("s"),
    "t.VendorID = s.VendorID AND t.tpep_pickup_datetime = s.tpep_pickup_datetime")
   .whenMatchedUpdateAll().execute())
print("V2: MERGE 1000 rows")

# V3: DELETE
spark.sql(f"DELETE FROM delta.`{TABLE}` WHERE passenger_count = 0")
print("V3: DELETE passenger_count=0")

# V4: Corrections
df_jan.limit(100).withColumn("fare_amount", lit(99.99)).write.format("delta").mode("append").save(TABLE)
print("V4: Append 100 corrections")

# V5: OPTIMIZE
dt.optimize().executeCompaction()
print("V5: OPTIMIZE")

# History
DeltaTable.forPath(spark, TABLE).history().select("version","timestamp","operation","operationMetrics").show(10, False)

# Time travel
for v in [0, 2, 3]:
    cnt = spark.read.format("delta").option("versionAsOf", v).load(TABLE).count()
    print(f"  VERSION {v}: {cnt:,} rows")

# Vacuum
files_before = glob.glob(f"{TABLE}/**/*.parquet", recursive=True)
size_before = sum(os.path.getsize(f) for f in files_before)
DeltaTable.forPath(spark, TABLE).vacuum(0)
files_after = glob.glob(f"{TABLE}/**/*.parquet", recursive=True)
size_after = sum(os.path.getsize(f) for f in files_after)
print(f"\nBefore VACUUM: {len(files_before)} files, {size_before/1e6:.1f} MB")
print(f"After  VACUUM: {len(files_after)} files, {size_after/1e6:.1f} MB")
print(f"Reclaimed: {(size_before-size_after)/1e6:.1f} MB ({(1-size_after/size_before)*100:.0f}%)")

try:
    spark.read.format("delta").option("versionAsOf", 0).load(TABLE).count()
except Exception as e:
    print(f"\nExpected error: {type(e).__name__}")
    print("REASON: VACUUM removed the Parquet files that V0 referenced.")
    print("Delta log JSON entries still exist, but physical .parquet files are gone.")
    print("LESSON: Never run VACUUM(0) in production. Use retentionHours=168 (7 days).")
```

---

## Solution 7.2 — Pipeline Monitor with Data Activator Simulation

```python
# solution_7_2_monitor.py
import duckdb, random, json, statistics
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import Optional, List, Callable
from collections import Counter

@dataclass
class PipelineRun:
    pipeline_id: str
    start_time: datetime
    end_time: Optional[datetime]
    status: str
    rows_processed: int
    error_message: Optional[str] = None

    @property
    def duration_minutes(self):
        return (self.end_time - self.start_time).total_seconds() / 60 if self.end_time else None

@dataclass
class ReflexTrigger:
    name: str
    condition: Callable[[PipelineRun], bool]
    cooldown_minutes: int = 30
    _last_fired: dict = field(default_factory=dict)

    def evaluate(self, run: PipelineRun) -> bool:
        last = self._last_fired.get(run.pipeline_id, datetime.min)
        if (run.start_time - last).total_seconds() / 60 < self.cooldown_minutes:
            return False
        if self.condition(run):
            self._last_fired[run.pipeline_id] = run.start_time
            return True
        return False

TRIGGERS = [
    ReflexTrigger("on_failure", lambda r: r.status == "FAILED"),
    ReflexTrigger("on_sla_breach", lambda r: (r.duration_minutes or 0) > 30),
    ReflexTrigger("on_low_row_count", lambda r: r.status == "SUCCESS" and r.rows_processed < 1000),
]

con = duckdb.connect(":memory:")
con.execute("""CREATE TABLE monitoring_events (
    pipeline_id VARCHAR, trigger_name VARCHAR,
    fired_at TIMESTAMP, run_status VARCHAR, alert_json VARCHAR
)""")

def fire_action(run: PipelineRun, trigger_name: str):
    card = {"alert": trigger_name, "pipeline": run.pipeline_id,
            "status": run.status, "rows": run.rows_processed,
            "duration_min": round(run.duration_minutes or 0, 1),
            "error": run.error_message or "—"}
    print(f"  [REFLEX → TEAMS] {json.dumps(card)}")
    con.execute("INSERT INTO monitoring_events VALUES (?,?,?,?,?)",
                [run.pipeline_id, trigger_name, datetime.utcnow(), run.status, json.dumps(card)])

class PipelineMonitor:
    def __init__(self): self.runs: List[PipelineRun] = []
    def record(self, run: PipelineRun):
        self.runs.append(run)
        for t in TRIGGERS:
            if t.evaluate(run):
                print(f"\n[TRIGGER FIRED] {t.name} | pipeline={run.pipeline_id}")
                fire_action(run, t.name)
    def report(self):
        total = len(self.runs)
        success = sum(1 for r in self.runs if r.status == "SUCCESS")
        failed  = [r for r in self.runs if r.status == "FAILED"]
        durations = [r.duration_minutes for r in self.runs if r.duration_minutes]
        return {
            "total_runs": total,
            "success_rate_pct": round(success/total*100, 1),
            "failed_runs": len(failed),
            "avg_duration_min": round(statistics.mean(durations), 2) if durations else 0,
            "most_failing_pipeline": Counter(r.pipeline_id for r in failed).most_common(1),
            "sla_compliant_pct": round(sum(1 for d in durations if d<=30)/len(durations)*100,1) if durations else 100,
        }

monitor = PipelineMonitor()
pipelines = ["ingest_taxi", "ingest_crimes", "silver_transform", "gold_agg", "report_refresh"]
base = datetime(2024, 1, 15)

for i in range(50):
    pid = random.choice(pipelines)
    start = base + timedelta(hours=i * 0.5)
    dur = random.uniform(5, 60)
    status = random.choices(["SUCCESS", "FAILED", "CANCELLED"], weights=[0.70, 0.25, 0.05])[0]
    monitor.record(PipelineRun(
        pipeline_id=pid, start_time=start,
        end_time=start + timedelta(minutes=dur),
        status=status,
        rows_processed=random.randint(0, 5_000_000) if status == "SUCCESS" else 0,
        error_message="TimeoutException: Spark job exceeded limit" if status == "FAILED" else None,
    ))

print("\n=== MONITORING REPORT ===")
for k, v in monitor.report().items(): print(f"  {k}: {v}")

print("\n=== ALERT LOG ===")
print(con.execute("SELECT pipeline_id, trigger_name, run_status, fired_at FROM monitoring_events").df().to_string(index=False))

# Data Activator conceptual config
REFLEX_CONFIG = {
    "triggers": [
        {"name": "on_failure", "condition": {"field": "status", "operator": "eq", "value": "FAILED"},
         "action": {"type": "teams_notification", "channel": "#data-alerts"}, "cooldown_minutes": 30},
        {"name": "on_sla_breach", "condition": {"field": "duration_minutes", "operator": "gt", "value": 30},
         "action": {"type": "email", "to": "dataops@company.com"}, "cooldown_minutes": 60},
    ]
}
print("\nData Activator Reflex Config:")
print(json.dumps(REFLEX_CONFIG, indent=2))
```

---

## Exam Quick Reference

| Scenario | Answer |
|---|---|
| Query Lakehouse via SQL, no writes needed | **SQL Analytics Endpoint** (read-only) |
| Full DML (INSERT/UPDATE/DELETE) via T-SQL | **Fabric Warehouse** |
| Virtualize S3/ADLS data without copying | **OneLake Shortcut** |
| Near-real-time sync from Azure SQL → Fabric | **Mirroring** |
| Power BI reads Delta without Import/DirectQuery | **DirectLake** mode |
| Alert on pipeline failure at 3 AM | **Data Activator (Reflex)** trigger |
| Morning ETL causing capacity throttling | Upgrade **SKU** or spread workloads across hours |
| Delta table storage growing uncontrolled | Schedule weekly `VACUUM(168)` |
| Power BI slow on 200M-row unpartitioned table | **Partition** + **Z-Order** + **OPTIMIZE** + DirectLake |
| Prevent duplicate rows on incremental load | Use **MERGE** with business key, not APPEND |
| Deploy notebook from Dev to Prod | **Deployment Pipeline** with stage-specific config |
| Version control for Fabric workspace items | **Git integration** (Azure DevOps / GitHub) |
