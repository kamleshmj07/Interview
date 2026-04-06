# DP-700: Practical Exercises for Senior Data Engineers
> Aligned to the Microsoft Fabric DP-700 Exam syllabus  
> Difficulty scale: 🟢 Basic → 🟡 Intermediate → 🔴 Advanced  
> All data sources are free, public, and accessible without authentication unless noted.

---

## Data Sources Reference

| Alias | URL | Format | Description |
|---|---|---|---|
| `NYC_TAXI` | https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2023-01.parquet | Parquet | NYC Yellow Taxi Trips (Jan 2023, ~3M rows) |
| `NYC_TAXI_CSV` | https://data.cityofnewyork.us/api/views/t29m-gskq/rows.csv?accessType=DOWNLOAD | CSV | NYC Taxi & Limousine trip records |
| `CHICAGO_CRIMES` | https://data.cityofchicago.org/api/views/ijzp-q8t2/rows.csv?accessType=DOWNLOAD | CSV | Chicago crimes dataset (~7M rows) |
| `WORLD_BANK_GDP` | https://api.worldbank.org/v2/en/indicator/NY.GDP.MKTP.CD?downloadformat=csv | CSV | World Bank GDP by country |
| `STOCK_SAMPLE` | https://raw.githubusercontent.com/plotly/datasets/master/finance-charts-apple.csv | CSV | Apple OHLCV stock price data |
| `IMDB_MOVIES` | https://datasets.imdbws.com/title.basics.tsv.gz | TSV.GZ | IMDB movie metadata |
| `GITHUB_EVENTS` | https://data.gharchive.org/2024-01-15-{0..23}.json.gz | JSON.GZ | GitHub Events stream (hourly) |
| `RETAIL_ORDERS` | https://raw.githubusercontent.com/microsoft/sql-server-samples/master/samples/databases/wide-world-importers/csv-files/orders.csv | CSV | Wide World Importers orders |

---

# MODULE 1: Workspace, Admin & Governance

---

## Exercise 1.1 — OneLake Architecture Setup 🟢

**Concept:** OneLake is the single logical data lake underpinning all Fabric items. Understanding its folder hierarchy, Lakehouse vs. Warehouse endpoints, and the role of the Delta format is foundational.

**Background:**  
Every Fabric workspace has one OneLake namespace. Lakehouses expose two endpoints: a Spark/file endpoint (`/Tables/` and `/Files/`) and a read-only SQL Analytics Endpoint. You will simulate this architecture locally using Delta Lake (open-source) before applying it in Fabric.

**Objective:**  
Design and populate a simulated OneLake structure locally using PySpark + Delta Lake representing three Lakehouses: `Bronze_LH`, `Silver_LH`, and `Gold_LH`.

**Setup (local):**
```bash
pip install pyspark delta-spark requests
```

**Tasks:**

1. Download the NYC Taxi Parquet file programmatically using Python `requests`.
2. Read the raw Parquet into a PySpark DataFrame. Print the schema and row count.
3. Write the raw data to a Delta table simulating `Bronze_LH/Tables/yellow_taxi_raw` with `overwrite` mode.
4. Inspect the Delta transaction log (`_delta_log/`) and describe what `00000000000000000000.json` contains.
5. Read the Bronze table back, apply schema validation (assert all expected columns exist, drop nulls on `tpep_pickup_datetime`), and write to `Silver_LH/Tables/yellow_taxi_cleaned`.
6. Confirm that `Silver_LH/Tables/yellow_taxi_cleaned` is a valid Delta table by running `DeltaTable.forPath(spark, path).history().show()`.
7. **Discussion question:** In real Fabric, what is the SQL Analytics Endpoint and why can't you run `INSERT` statements against it?

**Data Source:** `NYC_TAXI`  
**Skills:** PySpark, Delta Lake, Medallion Architecture

---

## Exercise 1.2 — Capacity Smoothing & Throttling Simulation 🟡

**Concept:** Fabric capacity uses a "smoothing" algorithm — burst usage above the capacity limit is borrowed against the next 24-hour window. If the debt exceeds a threshold, throttling kicks in (interactive operations are delayed or rejected).

**Background:**  
The Fabric Capacity Metrics App exposes CU (Capacity Unit) consumption data. You cannot run this locally, but you can simulate and analyze the throttling decision logic using Python.

**Objective:**  
Write a Python simulator that models Fabric capacity consumption and determines when throttling would trigger.

**Tasks:**

1. Create a Python dataclass `CapacityEvent` with fields: `timestamp: datetime`, `operation: str`, `cu_seconds: float`.
2. Generate 500 synthetic events over a 24-hour window. Assign heavier CU usage to "morning" hours (8–11 AM) simulating ETL pipeline runs.
3. Implement a `SmoothingEngine` class that:
   - Accepts a `capacity_sku` parameter (e.g., F4 = 4 CU, F8 = 8 CU).
   - Computes the rolling 10-minute smoothed CU consumption.
   - Tracks "carry-forward debt" when smoothed usage exceeds capacity.
   - Flags events as `THROTTLED` when accumulated debt exceeds 10 minutes of full capacity.
4. Plot the CU consumption timeline (matplotlib) showing: raw usage, smoothed usage, capacity limit line, and throttling periods highlighted in red.
5. Add a `generate_report()` method that outputs: total CU consumed, peak CU window, number of throttled operations, and estimated overage percentage.
6. **Challenge:** Parameterize the simulator to test F4 vs F8 vs F16 SKUs and show which SKU would have eliminated throttling for your synthetic workload.

**Skills:** Python dataclasses, time-series simulation, capacity planning

---

## Exercise 1.3 — Implementing Data Mesh with Domains 🔴

**Concept:** A Data Mesh in Fabric is implemented using two primitives: (1) **Domains** — logical groupings of workspaces aligned to business units, and (2) **Shortcuts** — pointers in OneLake that virtualize data from another Lakehouse, ADLS, S3, or GCS without copying it.

**Background:**  
In this exercise you model a company with two domains: `Finance` and `Operations`. The Finance domain owns the "source of truth" for transactions. Operations needs read access without owning the data.

**Tasks:**

1. Using Python + the Fabric REST API (documented at https://learn.microsoft.com/en-us/rest/api/fabric/core/), write a script outline (using `requests`) that would:
   - `POST /v1/admin/domains` to create a `Finance` domain.
   - `POST /v1/workspaces/{workspaceId}/assignToCapacity` to assign a workspace.
   - List the required OAuth2 scopes needed for these calls (research the Entra ID app registration docs).
2. Simulate the Shortcut concept locally: create two directory structures `finance_lh/Tables/transactions` and `operations_lh/Tables/` then write a Python function `create_shortcut(source_path, target_path)` that creates a symlink (on Linux/Mac) or junction (Windows) and validates the Delta table is readable from the target path.
3. Load the `RETAIL_ORDERS` CSV into the `finance_lh/Tables/transactions` Delta table using PySpark.
4. Read the data through the shortcut path (`operations_lh/Tables/transactions`) using PySpark and verify the row count matches exactly.
5. Simulate "Shortcut isolation" — modify the Delta table at the source path (append 100 new rows). Confirm the shortcut path reflects the new data immediately without any sync step.
6. Write a 300-word architecture decision record (ADR) explaining when you would choose a Shortcut vs. Mirroring vs. a full COPY for data sharing across domains.

**Skills:** REST API design, OneLake shortcuts, data mesh patterns, Delta Lake

---

# MODULE 2: Security & Governance

---

## Exercise 2.1 — Row-Level Security (RLS) Implementation 🟢

**Concept:** RLS restricts which rows a user can see. In Fabric Warehouse (T-SQL), this is implemented with Security Policies and inline table-valued functions. In Lakehouses (via SQL Endpoint), it can be done at the semantic model layer.

**Objective:**  
Implement RLS in a local DuckDB database (which supports SQL Security Policies) to simulate the Fabric Warehouse RLS pattern.

**Setup:**
```bash
pip install duckdb pandas
```

**Tasks:**

1. Download the `RETAIL_ORDERS` CSV and load it into a DuckDB table `orders`.
2. Add a column `region` to the table with values: `['APAC', 'EMEA', 'AMER']` assigned round-robin by `order_id % 3`.
3. Create a `users` mapping table: `(username VARCHAR, region VARCHAR)` with at least 3 users.
4. Write a T-SQL–style security predicate function (in DuckDB SQL): a view that filters `orders` based on `current_setting('app.current_user')` matched against the `users` table.
5. Simulate RLS by setting the session variable and querying:
   - As `user_apac` → only APAC rows should appear.
   - As `user_emea` → only EMEA rows.
   - As `admin` → all rows.
6. Measure the query performance impact of RLS (compare `EXPLAIN ANALYZE` output with and without the predicate filter).
7. Write the equivalent Fabric Warehouse DDL (`CREATE SECURITY POLICY`, `CREATE FUNCTION`) as SQL comments in your script — this is the syntax you must know for the exam.

**Data Source:** `RETAIL_ORDERS`  
**Skills:** T-SQL, RLS, DuckDB, security patterns

---

## Exercise 2.2 — Column-Level Security (CLS) and Dynamic Data Masking 🟡

**Concept:** CLS in Fabric Warehouse is achieved via `GRANT`/`DENY` on individual columns. Dynamic Data Masking (DDM) obscures data at query time — a user sees `XXXX-1234` instead of a full credit card number — without encrypting the underlying storage.

**Objective:**  
Implement both CLS and DDM patterns using DuckDB + Python to demonstrate the exam-relevant SQL syntax.

**Tasks:**

1. Create a `customers` table with columns: `customer_id`, `full_name`, `email`, `phone`, `ssn`, `salary`, `region`.
2. Populate with 10,000 synthetic rows using Python's `faker` library:
   ```bash
   pip install faker
   ```
3. Implement a masking function in Python that replicates DDM rules:
   - `email`: show only domain part (e.g., `****@gmail.com`)
   - `ssn`: show only last 4 digits (e.g., `XXX-XX-6789`)
   - `salary`: return `0` for non-finance roles
   - `phone`: full mask → `XXX-XXX-XXXX`
4. Create a `MaskedView` class that accepts a `role` parameter and applies appropriate masking when querying.
5. Write the **exact Fabric Warehouse T-SQL** for:
   - Granting SELECT on only `customer_id` and `region` to a role `ops_analyst`.
   - Denying SELECT on `ssn` and `salary` columns.
   - Creating a DDM mask on the `email` column using `Email()` mask function.
6. Demonstrate that a `DENY` on a column takes precedence over a table-level `GRANT` (document this behavior as a comment).

**Skills:** T-SQL DDL, data masking, security policy design, Faker

---

## Exercise 2.3 — Sensitivity Labels and Endorsement Pipeline 🔴

**Concept:** Sensitivity labels (via Microsoft Purview integration) classify data (e.g., `Confidential`, `Public`). Endorsement marks items as `Promoted` or `Certified` to signal quality. These are enforced via Information Protection policies.

**Objective:**  
Build a Python pipeline that programmatically classifies a dataset and generates an endorsement metadata report.

**Tasks:**

1. Download the `CHICAGO_CRIMES` dataset (first 50,000 rows, use the `$limit=50000` query parameter: `https://data.cityofchicago.org/api/views/ijzp-q8t2/rows.csv?accessType=DOWNLOAD&$limit=50000`).
2. Implement a `DataClassifier` class with these methods:
   - `scan_pii(df)` → returns a dict of `{column_name: pii_type}` by pattern-matching column names and sample values (look for email patterns, phone patterns, coordinates).
   - `suggest_label(pii_findings)` → returns one of `["Public", "General", "Confidential", "Highly Confidential"]` based on PII findings.
   - `generate_data_quality_score(df)` → returns a score 0–100 based on: null percentage, duplicate percentage, schema completeness, value range validity.
3. The Chicago crimes dataset has lat/lon columns, date columns, and description columns. Your scanner must flag lat/lon as `Location Data (PII-adjacent)`.
4. Generate an endorsement report as a JSON file containing: dataset name, suggested sensitivity label, data quality score, PII findings, column-level statistics, and recommended endorsement level (`None`, `Promoted`, or `Certified` — require score ≥ 85 for Certified).
5. Write a `EndorsementPolicy` class that enforces: a dataset cannot be `Certified` if it has `Highly Confidential` data without an approved exception flag.
6. **Exam tip exercise:** Write a short paragraph explaining the difference between a sensitivity label (set by Microsoft Purview) and endorsement (set by workspace contributors) — why can both exist on the same item simultaneously?

**Data Source:** `CHICAGO_CRIMES`  
**Skills:** PII detection, data classification, Python OOP, governance automation

---

# MODULE 3: Batch Ingestion & Transformation

---

## Exercise 3.1 — Full vs Incremental Load Pattern 🟢

**Concept:** A full load truncates and reloads all data on every run. An incremental (delta) load only processes new or changed records since the last run — tracked via a high-watermark column (e.g., `last_modified_date`).

**Objective:**  
Implement both patterns in PySpark against a Delta table and measure the difference in rows written.

**Tasks:**

1. Download the NYC Taxi Parquet file for January 2023 and February 2023:
   - Jan: `https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2023-01.parquet`
   - Feb: `https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2023-02.parquet`
2. **Full Load:** Write a `full_load(source_df, target_path)` function that writes Jan data to a Delta table using `mode("overwrite")`. Record the row count.
3. **Incremental Load:** Write an `incremental_load(source_df, target_path, watermark_col, last_watermark)` function that:
   - Filters `source_df` to only rows where `watermark_col > last_watermark`.
   - Appends filtered rows to the Delta table.
   - Persists the new high-watermark to a `_metadata/watermark.json` file.
4. Simulate Day 2: use Feb data as the "new batch." Run the incremental load using the watermark saved from the Jan run. Verify only Feb rows were appended.
5. Validate the final Delta table using `DeltaTable.history()` — you should see exactly 2 versions.
6. **Challenge:** Implement a `detect_late_arrivals(df, watermark_col, tolerance_hours=48)` function that flags rows whose timestamp is older than the current watermark minus 48 hours, and routes them to a `_late_arrivals/` Delta table instead of the main table.

**Data Source:** `NYC_TAXI` (Jan + Feb 2023)  
**Skills:** PySpark, Delta Lake, watermark pattern, incremental ingestion

---

## Exercise 3.2 — Dimensional Modeling: Star Schema with PySpark 🟡

**Concept:** Star schemas consist of a central Fact table surrounded by Dimension tables. In Fabric, the Gold Lakehouse layer typically holds the star schema consumed by Power BI via DirectLake.

**Objective:**  
Transform the raw NYC Taxi dataset into a proper Star Schema.

**Tasks:**

1. Load the Jan 2023 NYC Taxi Parquet into PySpark.
2. Build the following dimension tables:
   - `dim_date`: derived from `tpep_pickup_datetime` — columns: `date_key (int YYYYMMDD)`, `date`, `year`, `month`, `day`, `day_of_week`, `is_weekend`, `quarter`.
   - `dim_location`: load from the NYC Taxi Zone lookup CSV at `https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv` — columns: `location_key`, `borough`, `zone`, `service_zone`.
   - `dim_payment`: hardcoded mapping — `payment_type_id` → `payment_description` (1=Credit, 2=Cash, 3=No Charge, 4=Dispute, 5=Unknown).
   - `dim_vendor`: hardcoded — VendorID 1 = Creative Mobile, 2 = VeriFone.
3. Build `fact_trips`:
   - Keys: `date_key`, `pickup_location_key`, `dropoff_location_key`, `payment_type_id`, `vendor_id`.
   - Measures: `trip_distance`, `fare_amount`, `tip_amount`, `total_amount`, `passenger_count`, `trip_duration_minutes` (computed).
4. Write all tables as Delta format to `gold_lh/Tables/` with appropriate save modes.
5. Write a DuckDB SQL query that joins all dimension and fact tables to answer: "Which borough had the highest average fare on weekends in January 2023?"
6. **Challenge:** Implement a `dim_date` generator as a pure Python function (no Spark) that generates all dates for a given year using `pandas`. This mirrors what you'd do in a Dataflow Gen2 M query.

**Data Source:** `NYC_TAXI`, NYC Zone Lookup CSV  
**Skills:** PySpark, dimensional modeling, star schema, Delta Lake

---

## Exercise 3.3 — PySpark Deduplication and Data Cleaning (Silver Layer) 🟡

**Concept:** The Silver layer applies business rules: deduplication, null handling, type casting, and standardization. In Fabric Notebooks, this runs on Spark pools.

**Objective:**  
Build a robust Silver layer transformation pipeline for the Chicago Crimes dataset.

**Tasks:**

1. Download the Chicago Crimes dataset (use `$limit=100000`).
2. Load into PySpark. Identify and document:
   - Columns with >10% null values.
   - Duplicate rows (exact duplicates by `ID` column).
   - Type inconsistencies (e.g., `Date` is a string — parse it to `TimestampType`).
3. Implement a `SilverTransformer` class with these PySpark transformation methods:
   - `drop_duplicates(df, key_col="ID")` → keeps the latest row per key.
   - `handle_nulls(df)` → fills categorical nulls with `"UNKNOWN"`, numeric nulls with `0`, drops rows where `Date` is null.
   - `cast_types(df)` → parses `Date` string to Timestamp, casts `Latitude`/`Longitude` to DoubleType.
   - `standardize_text(df, cols)` → uppercases and strips whitespace from specified string columns.
   - `add_audit_columns(df)` → adds `ingestion_timestamp`, `source_system`, `batch_id` columns.
4. Chain all transformations using method chaining or a `transform()` pipeline pattern.
5. Write the final cleaned DataFrame to `silver_lh/Tables/chicago_crimes_cleaned` as Delta.
6. Run `DESCRIBE HISTORY` on the Delta table and explain what each `operationParameters` key means.
7. **Advanced:** Implement a `DataQualityReport` that runs after transformation and asserts: zero duplicates on `ID`, null rate per column < 5% for critical columns, row count within 5% of input row count.

**Data Source:** `CHICAGO_CRIMES`  
**Skills:** PySpark, data cleaning, Delta Lake, data quality assertions

---

## Exercise 3.4 — MERGE (Upsert) Pattern in T-SQL and PySpark 🔴

**Concept:** `MERGE` (also called upsert) is the most exam-critical SQL operation. It updates existing rows and inserts new ones atomically. In Fabric Warehouse it's T-SQL `MERGE`. In Lakehouses it's Delta Lake `MERGE INTO` via PySpark or Spark SQL.

**Objective:**  
Implement MERGE in both T-SQL (DuckDB) and PySpark Delta Lake and compare their behavior.

**Tasks:**

1. Create a `dim_customer` target table with 1000 rows using `faker`.
2. Create an `incoming_updates` DataFrame simulating a CDC (Change Data Capture) feed:
   - 200 rows that UPDATE existing customers (changed email or phone).
   - 100 rows that are INSERTs (new `customer_id`s).
   - 50 rows that should trigger a SOFT DELETE (set `is_active = False`).
3. **DuckDB T-SQL MERGE:**
   ```sql
   MERGE INTO dim_customer AS target
   USING incoming_updates AS source
   ON target.customer_id = source.customer_id
   WHEN MATCHED AND source.operation = 'DELETE' THEN UPDATE SET is_active = False
   WHEN MATCHED THEN UPDATE SET email = source.email, phone = source.phone, updated_at = NOW()
   WHEN NOT MATCHED THEN INSERT (...) VALUES (...);
   ```
   Execute and verify counts of updated, inserted, and soft-deleted rows.
4. **PySpark Delta MERGE:**
   ```python
   from delta.tables import DeltaTable
   delta_table.alias("target").merge(
       source_df.alias("source"),
       "target.customer_id = source.customer_id"
   ).whenMatchedUpdate(...).whenNotMatchedInsert(...).execute()
   ```
   Execute and verify using `DeltaTable.history()`.
5. Compare: run the same logical operation twice (idempotency test). The second run should produce 0 changes. Verify this for both implementations.
6. **SCD Type 2:** Extend the DuckDB implementation to track history — instead of updating in-place, insert a new row with `valid_from` and `valid_to` dates, and set `is_current = True` on the new row and `False` on the old.
7. Write the equivalent **Fabric Warehouse CTAS** pattern:
   ```sql
   CREATE TABLE dim_customer_new AS
   SELECT ... FROM dim_customer ...
   ```
   Explain when you'd prefer CTAS over MERGE and the trade-offs in Fabric Warehouse.

**Skills:** T-SQL MERGE, Delta Lake MERGE, SCD Type 2, DuckDB, PySpark

---

## Exercise 3.5 — Data Factory Pipeline: Parameterized and Branching Logic 🔴

**Concept:** Data Factory pipelines in Fabric support: `ForEach`, `If Condition`, `Set Variable`, `Execute Pipeline`, and `Web Activity`. Dynamic expressions use `@pipeline().parameters.paramName` syntax.

**Objective:**  
Design a parameterized, branching ingestion pipeline and implement the orchestration logic in Python (simulating what the pipeline JSON would do).

**Tasks:**

1. Define a `PipelineConfig` dataclass:
   ```python
   @dataclass
   class PipelineConfig:
       source_type: str  # "csv" | "parquet" | "json"
       source_url: str
       target_table: str
       load_mode: str   # "full" | "incremental"
       watermark_col: Optional[str]
   ```
2. Create a `PipelineOrchestrator` class that:
   - Accepts a list of `PipelineConfig` objects.
   - Routes each config to the correct reader (`read_csv`, `read_parquet`, `read_json`).
   - Applies the correct load mode (full or incremental) from Exercise 3.1.
   - Catches per-pipeline errors and continues processing remaining pipelines (don't fail the whole batch).
   - Logs each pipeline's status: `SUCCESS`, `FAILED`, or `SKIPPED`.
3. Define three pipeline configs using your datasets: one CSV (Chicago Crimes), one Parquet (NYC Taxi), one JSON (use `https://jsonplaceholder.typicode.com/posts` as a lightweight JSON source).
4. Run the orchestrator and produce a run summary report (total, succeeded, failed).
5. Implement retry logic: failed pipelines retry up to 3 times with exponential backoff (`2^attempt` seconds).
6. Translate your `PipelineOrchestrator` design into a pseudo-JSON pipeline definition matching Fabric Data Factory's actual JSON schema structure (activities array, dependsOn, typeProperties). You don't need to run it — write it as a Python dict and `json.dumps` it.

**Skills:** Python orchestration, retry logic, pipeline design, Data Factory concepts

---

# MODULE 4: Real-Time Intelligence

---

## Exercise 4.1 — KQL Fundamentals 🟢

**Concept:** Kusto Query Language (KQL) is a read-only query language optimized for time-series and log analytics. Core operators: `where`, `summarize`, `extend`, `project`, `join`, `render`.

**Objective:**  
Practice KQL patterns using a local Kusto emulator or by translating KQL into equivalent DuckDB SQL.

**Setup:**  
Download the KQL CLI (free): `https://aka.ms/kusto-linux` or use the Azure Data Explorer free cluster at `https://dataexplorer.azure.com/clusters/help` (no auth required for the `help` cluster).

**Tasks:**

1. Connect to the `help` cluster → `Samples` database → `StormEvents` table (pre-populated).
2. Write KQL for each of the following:
   - Count events by `EventType`, sorted descending, top 10.
   - Filter storms in `State == "FLORIDA"` between June and August 2007.
   - Compute average `DamageProperty` per `EventType` where count > 100.
   - Use `extend` to add a column `is_severe = DamageProperty > 1000000`.
   - `render` a timechart of daily storm counts.
3. Download the Apple stock CSV (`STOCK_SAMPLE`) and load it into a local pandas DataFrame. Translate the following KQL pattern into pandas:
   - `| summarize avg_close = avg(AAPL_close) by bin(Date, 7d)` → weekly average close price.
   - `| where AAPL_close < prev(AAPL_close)` → days where price dropped.
4. Write the KQL `| make-series` equivalent for creating a time series of daily stock prices and applying anomaly detection with `series_decompose_anomalies()` — then implement the same anomaly detection logic in Python using `statsmodels`.

**Skills:** KQL, time-series queries, pandas translation, Azure Data Explorer

---

## Exercise 4.2 — Windowing Functions: Tumbling, Hopping, and Sliding 🟡

**Concept:**  
- **Tumbling window:** Fixed-size, non-overlapping (e.g., every 5 minutes).  
- **Hopping window:** Fixed-size but overlapping (e.g., 10-minute window, advancing every 5 minutes).  
- **Sliding window:** Emits output on every event, window = last N seconds from event time.

These are used in Fabric's Real-Time Intelligence (Eventstreams + KQL) and are conceptually equivalent to stream processing.

**Objective:**  
Implement all three windowing patterns in Python using a simulated event stream (stock price ticks).

**Tasks:**

1. Generate a synthetic stream of stock price events: 10,000 events over 2 hours, timestamps spaced 0.5–2 seconds apart randomly, prices following a random walk starting at $100.
2. **Tumbling Window (5 min):**
   - Aggregate: `open`, `high`, `low`, `close`, `volume` (simulate volume as random int 100–1000).
   - Output: one OHLCV bar per 5-minute bucket. This is exactly a candlestick chart.
3. **Hopping Window (10 min window, 5 min hop):**
   - Compute `avg_price` and `price_std` per window.
   - Each event falls in exactly 2 overlapping windows — verify this.
4. **Sliding Window (last 5 min from each event):**
   - For each event, compute `rolling_avg` and flag if price has dropped more than 10% from the window's opening price.
   - This is the "stock price drop alert" scenario from the study plan.
5. For the sliding window drop alerts, implement a `trigger_alert(event, drop_pct)` function that would (in real Fabric) send a Reflex/Data Activator notification. Simulate by printing a structured alert dict.
6. **KQL Translation:** Write the KQL query using `bin()` that replicates your tumbling window aggregation. Write it as a string in Python.

**Skills:** Stream processing, windowing patterns, time-series, KQL concepts

---

## Exercise 4.3 — KQL Advanced: Materialized Views and Update Policies 🔴

**Concept:**  
- **Materialized Views:** Pre-computed aggregations in KQL that update incrementally as new data lands. Defined with `.create materialized-view`.  
- **Update Policies:** Automatically transform data as it's ingested into a table, routing it to a derived table.

**Objective:**  
Simulate Materialized View and Update Policy behavior in Python, and write the exact KQL DDL commands.

**Tasks:**

1. Using the Azure Data Explorer free cluster (`https://dataexplorer.azure.com/clusters/help/databases/Samples`), write and execute:
   - A KQL `summarize` query that would be the basis of a materialized view on `StormEvents`: hourly event count by state.
   - Write the `.create materialized-view` command (even if you don't have write access to this cluster, write the DDL as a comment and explain each parameter: `lookback`, `autoUpdateSchema`, `docString`).
2. Locally simulate the Update Policy pattern:
   - Implement a `raw_events` list that receives incoming JSON events.
   - Implement an `update_policy(event)` function that transforms each event (parse ISO timestamp, extract subdomain from URL, categorize by event type).
   - Automatically route transformed events to `processed_events` list.
   - This simulates how a KQL Update Policy would transform data from `RawTable` to `ProcessedTable` on ingestion.
3. Implement a `MaterializedViewSimulator` class:
   - Maintains a pre-computed aggregation dict: `{(state, hour_bucket): count}`.
   - On each new event, updates only the affected bucket (incremental update — don't recompute all).
   - Exposes a `query(state, start_hour, end_hour)` method that reads from the pre-computed state.
   - Compare query latency: reading from pre-computed state vs. scanning all raw events.
4. Benchmark: generate 100,000 events, then run 1,000 random queries. Compare total query time with and without the materialized view simulation. Compute speedup factor.
5. Write the KQL `.create table`, `.create materialized-view`, and `.alter table policy update` DDL commands as Python string constants with inline comments explaining each line.

**Skills:** KQL DDL, materialized views, update policies, performance benchmarking

---

# MODULE 5: Optimization

---

## Exercise 5.1 — V-Order and Partitioning for Lakehouse Performance 🟡

**Concept:**  
- **V-Order:** A Microsoft-proprietary write optimization applied to Parquet/Delta files that sorts and encodes data for faster Power BI (DirectLake) reads. Enabled by default in Fabric.  
- **Partitioning:** Physically organizes Delta files into subdirectories by a column value (e.g., `/year=2023/month=01/`), enabling partition pruning.

**Objective:**  
Benchmark query performance with and without partitioning, and understand V-Order's effect.

**Tasks:**

1. Load both months of NYC Taxi data (Jan + Feb 2023) into a single PySpark DataFrame (~6M rows).
2. **Unpartitioned write:** Write to `delta_unpartitioned/` with no partitioning. Record file count and write time.
3. **Partitioned write:** Write to `delta_partitioned/` with `.partitionBy("year", "month")` after adding year/month columns from `tpep_pickup_datetime`. Record file count and write time.
4. Run the following query against both tables and compare execution time using `time.perf_counter()`:
   ```python
   df.filter((col("year") == 2023) & (col("month") == 1))
     .groupBy("PULocationID")
     .agg(avg("total_amount").alias("avg_fare"))
   ```
5. Inspect the Delta log for the partitioned table. Identify which `.parquet` files would be skipped (partition pruning) when filtering for only January data.
6. **V-Order simulation:** PySpark open-source doesn't support V-Order natively (it's Fabric-only). Instead, simulate the concept by writing data sorted by the most frequently filtered column:
   ```python
   df.sort("tpep_pickup_datetime").write.format("delta")...
   ```
   Measure whether pre-sorted writes improve subsequent filter query latency.
7. Write a performance comparison table (markdown) documenting: write time, file count, query time, and data scanned for all three approaches.

**Data Source:** `NYC_TAXI` (Jan + Feb)  
**Skills:** Delta Lake partitioning, PySpark optimization, performance benchmarking

---

## Exercise 5.2 — Z-Order Clustering for Multidimensional Filtering 🟡

**Concept:** Z-Order (also called Z-order curve or Morton code) sorts data along multiple dimensions simultaneously, ensuring data with similar values across multiple columns is co-located in the same files. It improves performance when filtering on multiple non-partition columns.

**Objective:**  
Implement Z-Order optimization using Delta Lake's `OPTIMIZE ZORDER BY` command and benchmark multi-column filter queries.

**Tasks:**

1. Load the Chicago Crimes dataset (100K rows) into a Delta table WITHOUT Z-Order. Time a query filtering on `PrimaryType = 'THEFT'` AND `Year = 2020`.
2. Apply Z-Order optimization:
   ```python
   from delta.tables import DeltaTable
   dt = DeltaTable.forPath(spark, path)
   dt.optimize().executeZOrderBy(["PrimaryType", "Year", "District"])
   ```
3. Re-run the same filter query and compare execution time. Note: with only 100K rows the difference may be small — document this and explain at what data scale Z-Order becomes significant (include the theoretical justification).
4. Inspect the Delta log `_delta_log/` after OPTIMIZE — identify the `REMOVE` operations (old files) and `ADD` operations (new compacted files). What is the new file count vs. original?
5. Combine Z-Order with partitioning: partition by `Year`, then Z-Order by `PrimaryType` and `District` within each partition. Document the expected query plan benefit.
6. Explain in writing: when should you use Z-Order vs. partitioning? Include the rule of thumb about partition cardinality (too many partitions = too many small files).

**Data Source:** `CHICAGO_CRIMES`  
**Skills:** Delta Lake OPTIMIZE, Z-Order, performance analysis

---

## Exercise 5.3 — Delta Table Maintenance: Vacuum and History 🔴

**Concept:**  
- **VACUUM:** Removes old Parquet files that are no longer referenced by the Delta transaction log. By default, files older than 7 days are retained for time travel. Running `VACUUM` with a shorter retention period reclaims storage but eliminates time travel for that window.  
- **Time Travel:** Delta Lake lets you query historical versions using `VERSION AS OF` or `TIMESTAMP AS OF`.

**Objective:**  
Build a complete Delta table lifecycle management script.

**Tasks:**

1. Create a Delta table from the NYC Taxi Jan data.
2. Simulate 5 days of updates:
   - Day 1: Append Feb data.
   - Day 2: Run a MERGE to update 1,000 rows (change `total_amount` by +5%).
   - Day 3: Delete all rows where `passenger_count == 0`.
   - Day 4: Append a small "corrections" batch (100 rows with fixed values).
   - Day 5: Run OPTIMIZE.
3. After all operations, run `DESCRIBE HISTORY` — document all 6 versions (0–5) and what operation each represents.
4. **Time Travel queries:**
   - Query the table `VERSION AS OF 0` (original raw data) — get row count.
   - Query `VERSION AS OF 2` (after MERGE) — verify the modified `total_amount` rows.
   - Query the state just before the deletes: `VERSION AS OF 2` vs `VERSION AS OF 3`.
5. Calculate storage usage before and after `VACUUM`:
   - List all `.parquet` files in the Delta directory before VACUUM — record total size.
   - Run `VACUUM` with `retentionHours=0` (this requires `spark.databricks.delta.retentionDurationCheck.enabled=false`).
   - List remaining files. Document what percentage of storage was reclaimed.
6. **Exam trap:** After running VACUUM with 0-hour retention, attempt a time travel query to `VERSION AS OF 0`. Document the exact error message and explain why this happens. Write a recommended VACUUM schedule for a production Fabric Lakehouse.

**Skills:** Delta Lake VACUUM, time travel, table maintenance, storage optimization

---

## Exercise 5.4 — SQL Analytics Endpoint vs Warehouse: Query Tuning 🔴

**Concept:**  
- **SQL Analytics Endpoint:** Auto-generated, read-only T-SQL interface over a Lakehouse. Supports SELECT only.  
- **Fabric Warehouse:** Full DML (INSERT, UPDATE, DELETE, MERGE), DDL (CREATE TABLE, CREATE VIEW), and supports distributed query execution.  
- **Query Tuning:** In Warehouse, this involves: avoiding SELECT *, using `RESULT_SET_CACHING`, choosing correct distribution, and tuning JOIN order.

**Objective:**  
Write and optimize T-SQL queries against a DuckDB database (simulating Fabric Warehouse behavior) and document the equivalent Fabric-specific optimizations.

**Tasks:**

1. Load the NYC Taxi (Jan 2023) and Zone Lookup CSV into DuckDB tables.
2. Write a baseline "bad" query:
   ```sql
   SELECT * FROM yellow_taxi t, taxi_zones pu, taxi_zones do
   WHERE t.PULocationID = pu.LocationID AND t.DOLocationID = do.LocationID
   AND t.tpep_pickup_datetime > '2023-01-15'
   ```
3. Rewrite as an optimized query:
   - Use explicit `JOIN` syntax instead of comma-separated FROM.
   - `SELECT` only needed columns.
   - Add a filter that enables partition pruning (if partitioned).
   - Use a CTE for clarity.
4. Compare `EXPLAIN` plans for both queries. Identify: scan type (full vs. filter), join algorithm, estimated rows.
5. Write a window function query in T-SQL:
   ```sql
   SELECT vendor_id,
          tpep_pickup_datetime,
          total_amount,
          AVG(total_amount) OVER (PARTITION BY vendor_id ORDER BY tpep_pickup_datetime ROWS BETWEEN 99 PRECEDING AND CURRENT ROW) AS rolling_100_avg,
          RANK() OVER (PARTITION BY vendor_id ORDER BY total_amount DESC) AS fare_rank
   FROM yellow_taxi
   ```
   Execute and explain what `ROWS BETWEEN 99 PRECEDING AND CURRENT ROW` means.
6. Write the Fabric Warehouse equivalents as SQL comments:
   - How to enable result set caching: `SET RESULT_SET_CACHING ON`
   - How to check query history: `sys.dm_pdw_exec_requests`
   - Why Fabric Warehouse doesn't have clustered indexes (MPP architecture)
   - The CTAS pattern for materializing heavy queries

**Skills:** T-SQL query optimization, window functions, EXPLAIN plans, DuckDB

---

# MODULE 6: Lifecycle Management & ALM

---

## Exercise 6.1 — Git Integration and Schema Versioning 🟢

**Concept:** Fabric Git integration allows workspace items (notebooks, pipelines, SQL scripts) to be committed to Azure DevOps or GitHub. Each item is serialized as a folder containing JSON/YAML metadata files.

**Objective:**  
Simulate Fabric's Git integration model using a local Git repo.

**Tasks:**

1. Initialize a Git repo: `git init fabric_workspace`.
2. Create the folder structure mirroring Fabric's Git serialization format:
   ```
   fabric_workspace/
   ├── .platform/
   ├── Notebooks/
   │   └── Silver_Transform.Notebook/
   │       ├── notebook-content.py
   │       └── .platform
   ├── Pipelines/
   │   └── Ingest_NYC_Taxi.DataPipeline/
   │       └── pipeline-content.json
   └── Warehouses/
       └── Gold_Warehouse.Warehouse/
           └── definition.json
   ```
3. Create a `notebook-content.py` containing your Silver transformation code from Exercise 3.3.
4. Create a `pipeline-content.json` that represents a simple pipeline with one Copy Activity (use the structure from the Fabric documentation at `https://learn.microsoft.com/en-us/fabric/data-factory/pipeline-json-schema`).
5. Make 3 commits simulating a real development workflow:
   - Commit 1: Initial Bronze ingestion notebook.
   - Commit 2: Add Silver transformation.
   - Commit 3: Add Gold aggregation and pipeline.
6. Run `git log --oneline --graph` and `git diff HEAD~1 HEAD` — practice reading the diff output for notebook changes. In real Fabric, this is exactly what reviewers see in pull requests.
7. Create a `dev` branch, make a breaking schema change (rename a column), then write a migration script that handles the rename safely. Merge back to `main`.

**Skills:** Git, Fabric workspace structure, ALM, schema migration

---

## Exercise 6.2 — Deployment Pipeline Simulation: Dev → Test → Prod 🟡

**Concept:** Fabric Deployment Pipelines promote workspace content through Dev → Test → Prod stages. Each promotion copies workspace items while allowing stage-specific configurations (e.g., different data sources per environment).

**Objective:**  
Implement a deployment pipeline simulator in Python.

**Tasks:**

1. Define three environment configurations:
   ```python
   ENVIRONMENTS = {
       "dev":  {"lakehouse_path": "/tmp/dev_lh",  "spark_pool": "small",  "log_level": "DEBUG"},
       "test": {"lakehouse_path": "/tmp/test_lh", "spark_pool": "medium", "log_level": "INFO"},
       "prod": {"lakehouse_path": "/tmp/prod_lh", "spark_pool": "large",  "log_level": "WARNING"},
   }
   ```
2. Create a `DeploymentPipeline` class that:
   - Holds a list of `WorkspaceItem` objects (notebooks, pipelines, SQL scripts).
   - Has a `promote(source_env, target_env)` method.
   - During promotion: validates the item in `source_env` (runs a dry-run), copies the item config with target environment variables substituted, and creates a deployment record.
3. Implement pre-deployment validation:
   - Check that all Delta tables expected by notebooks exist in the source environment.
   - Check that no notebook has uncommitted changes (read `.git/status` programmatically).
   - Check that the target environment has sufficient "capacity" (simulate with a configurable threshold).
4. Implement a `RollbackManager` that can revert a failed promotion by restoring the previous deployment record.
5. Generate a deployment manifest JSON file for each promotion containing: timestamp, promoted items, source/target environments, validation results, and operator (use `os.getenv("USER")`).
6. **Exam scenario:** Document the key differences between: (a) deploying a Notebook via Deployment Pipeline vs. (b) pushing a notebook via Git sync — in terms of what gets updated, how parameters are handled, and conflict resolution.

**Skills:** Python, deployment automation, environment management, ALM patterns

---

# MODULE 7: Monitoring

---

## Exercise 7.1 — Spark Job Monitoring and DAG Analysis 🟡

**Concept:** The Spark UI shows the DAG (Directed Acyclic Graph) of stages and tasks for each job. Key metrics: stage duration, task distribution, shuffle read/write, GC time. In Fabric, this is accessible via the Monitoring Hub.

**Objective:**  
Instrument a PySpark job to capture and analyze its own execution metrics.

**Tasks:**

1. Write a PySpark job that performs a complex aggregation on the Chicago Crimes dataset:
   - Join with a self-created `district_metadata` table.
   - Group by 3 columns, compute 5 aggregates.
   - Write to Delta.
2. Instrument the job using Spark's `SparkListener` API (Python):
   ```python
   from pyspark import SparkContext
   sc = SparkContext.getOrCreate()
   # Register a custom listener to capture stage metrics
   ```
   Capture: `stageCompleted` events with `stageInfo.taskMetrics`.
3. Extract and print: stage ID, number of tasks, total execution time, shuffle bytes read, shuffle bytes written, peak JVM memory.
4. Identify the most expensive stage (by execution time). Look at the shuffle metrics — if shuffle read > 0, explain which operation caused the shuffle (groupBy? join? sort?).
5. Implement a `SparkJobProfiler` context manager:
   ```python
   with SparkJobProfiler("silver_transform_job") as profiler:
       # run spark operations
       ...
   profiler.report()  # prints formatted metrics table
   ```
6. **Optimization loop:** After profiling, apply one optimization (e.g., broadcast join for the small `district_metadata` table, cache intermediate result, repartition to reduce shuffle). Re-profile and document the improvement.

**Skills:** PySpark monitoring, SparkListener, performance profiling, optimization

---

## Exercise 7.2 — Pipeline Failure Handling and Data Activator Simulation 🔴

**Concept:** Data Activator (Reflex) monitors data streams and triggers actions (email, Teams message, pipeline run) when conditions are met. In Fabric, you define `Triggers` (conditions) and `Actions` (responses).

**Objective:**  
Build a complete pipeline monitoring and alerting system simulating Data Activator's behavior.

**Tasks:**

1. Create a `PipelineMonitor` class that tracks pipeline runs:
   ```python
   @dataclass
   class PipelineRun:
       pipeline_id: str
       start_time: datetime
       end_time: Optional[datetime]
       status: str   # RUNNING, SUCCESS, FAILED, CANCELLED
       rows_processed: int
       error_message: Optional[str]
   ```
2. Implement `ReflexTrigger` — a class that evaluates conditions on a stream of `PipelineRun` objects:
   - `trigger_on_failure(pipeline_id)` → fires if status == FAILED.
   - `trigger_on_sla_breach(pipeline_id, max_duration_minutes)` → fires if run exceeds max duration.
   - `trigger_on_row_count_anomaly(pipeline_id, expected_min_rows)` → fires if `rows_processed < expected_min_rows`.
3. Implement `ReflexAction` — a class with pluggable action handlers:
   - `send_teams_notification(message)` → simulate by printing a formatted Teams card JSON.
   - `trigger_remediation_pipeline(pipeline_id)` → simulate by logging and calling `PipelineOrchestrator.run(pipeline_id)` from Exercise 3.5.
   - `log_to_monitoring_table(run, trigger)` → write to a DuckDB `monitoring_events` table.
4. Generate 50 synthetic pipeline runs (mix of successes, failures, SLA breaches) and run the monitor against them.
5. Produce a monitoring report: total runs, success rate, MTTR (mean time to recovery), most frequently failing pipeline, and average SLA compliance rate.
6. **Exam extension:** Write the Data Activator Reflex configuration as a Python dict matching the conceptual schema (`trigger`, `condition`, `action`, `cooldown_minutes`) — this simulates what you'd configure in the Fabric Reflex UI.

**Skills:** Monitoring, alerting, Data Activator concepts, Python OOP, DuckDB

---

# MODULE 8: Comprehensive Integration Scenarios

---

## Exercise 8.1 — End-to-End Medallion Pipeline 🔴

**Concept:** The capstone exercise — build a complete Bronze → Silver → Gold pipeline simulating a production Fabric Lakehouse implementation.

**Objective:**  
Build an end-to-end, parameterized, monitored, idempotent data pipeline using all concepts from the course.

**Tasks:**

1. **Bronze Layer:** Ingest NYC Taxi Jan + Feb 2023 data and Chicago Crimes data as raw Delta tables with audit columns.
2. **Silver Layer:**
   - Deduplicate and clean each dataset using the `SilverTransformer` from Exercise 3.3.
   - Apply RLS metadata tags (add a `data_classification` column).
   - Run data quality assertions — fail the pipeline if quality score < 70.
3. **Gold Layer:**
   - Build the star schema from Exercise 3.2 for the taxi data.
   - Build a `fact_crime_summary` aggregated table: daily crime counts by district, crime type, and arrest status.
   - Partition both Gold tables appropriately.
4. **Orchestration:**
   - Use the `PipelineOrchestrator` from Exercise 3.5 to chain all three layers.
   - Each layer depends on the previous — implement a simple DAG with dependency checks.
5. **Monitoring:**
   - Use the `PipelineMonitor` from Exercise 7.2 to track each layer's execution.
   - Trigger an alert if any layer fails.
   - Write a final run report to a `pipeline_audit` Delta table.
6. **Lifecycle:**
   - All pipeline code should be committed to Git (from Exercise 6.1).
   - Write a `deploy.py` script that uses the `DeploymentPipeline` from Exercise 6.2 to promote from dev to test.
7. **Final output:** Generate a single JSON "data catalog" entry for the Gold layer describing: tables, schemas, data lineage (source → bronze → silver → gold), row counts, freshness timestamp, data quality score, and sensitivity label.

**Skills:** All modules combined — end-to-end data engineering

---

## Exercise 8.2 — Exam Simulation: Scenario-Based Questions 🔴

**Concept:** The DP-700 exam presents scenario-based questions requiring you to choose the correct Fabric tool, optimization, or configuration. These exercises sharpen that judgment.

**Objective:**  
For each scenario below, write a detailed technical justification (150–300 words) AND implement a code prototype.

**Scenarios:**

**Scenario A — Tool Selection:**  
Your team needs to ingest 500GB of CSV files from an on-premises SQL Server monthly. The files arrive at 2 AM and must be available in the Gold layer by 6 AM. Latency doesn't need to be below 1 hour.  
→ Question: Should you use (1) Dataflow Gen2, (2) Data Factory Copy Activity with a Gateway, or (3) a Fabric Notebook with PySpark + JDBC?  
→ Implement a connection test script for option (2) using `pyodbc` to verify the on-prem connectivity pattern.

**Scenario B — Optimization:**  
A Power BI report on a 200M-row Delta table is taking 45 seconds to load. The table is not partitioned. DirectLake mode is enabled. Users always filter by `region` and `product_category`.  
→ Question: What is your optimization strategy? In what order would you apply: Partitioning, Z-Order, V-Order, table OPTIMIZE, pre-aggregated Gold table?  
→ Implement a benchmarking script that tests filter query time with and without each optimization on a 1M-row sample.

**Scenario C — Security:**  
A new regulation requires that analysts in the EU workspace cannot see any rows where `country = 'US'` and cannot see the `ssn` column at all. Power BI reports must enforce this automatically.  
→ Question: What combination of RLS, CLS, and Sensitivity Labels would you implement? At which layer (Lakehouse, Warehouse, Semantic Model)?  
→ Write all required T-SQL DDL statements for the Warehouse implementation.

**Scenario D — Real-Time:**  
You need to alert operations when the average processing time of events in a manufacturing IoT stream exceeds 2 seconds for any 5-minute window.  
→ Question: Design the full Eventstream → KQL → Data Activator flow.  
→ Implement the windowing logic from Exercise 4.2 adapted to IoT latency data (generate synthetic data where `event_id`, `machine_id`, `processing_time_ms`, `timestamp`).

---

## Appendix: Quick Reference — Exam-Critical Syntax

### PySpark
```python
# Read Delta
df = spark.read.format("delta").load("abfss://...")

# Write Delta with partitioning
df.write.format("delta").mode("overwrite").partitionBy("year","month").save(path)

# Delta MERGE
from delta.tables import DeltaTable
DeltaTable.forPath(spark, path).alias("t").merge(
    source.alias("s"), "t.id = s.id"
).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()

# MSSparkUtils (Fabric only)
mssparkutils.fs.ls("abfss://...")
mssparkutils.credentials.getSecret("keyvault_name", "secret_name")
```

### T-SQL (Fabric Warehouse)
```sql
-- CTAS
CREATE TABLE gold.dim_customer AS SELECT ...;

-- MERGE
MERGE INTO target AS t USING source AS s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT (...) VALUES (...);

-- RLS
CREATE FUNCTION security.fn_rls(@region VARCHAR)
RETURNS TABLE WITH SCHEMABINDING AS RETURN SELECT 1 AS result
WHERE @region = USER_NAME();
CREATE SECURITY POLICY rls_policy ADD FILTER PREDICATE security.fn_rls(region) ON dbo.orders;

-- Window Functions
SELECT *, RANK() OVER (PARTITION BY vendor_id ORDER BY total_amount DESC) AS rnk FROM fact_trips;
```

### KQL
```kql
// Basic aggregation
StormEvents | summarize count() by EventType | top 10 by count_

// Time windowing
StormEvents | summarize events=count() by bin(StartTime, 1h), State | render timechart

// Join
Table1 | join kind=inner Table2 on $left.key == $right.key

// Materialized view DDL
.create materialized-view with (backfill=true) HourlyStats on StormEvents {
    StormEvents | summarize count() by bin(StartTime, 1h), State
}
```
