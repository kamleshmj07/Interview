# Data Engineering — Basic Concepts

How to use this: each concept has what it is, a concrete example of why it matters, the trade-offs/comparisons that interviewers actually probe, a text diagram where one helps, and a tight "interview-ready answer" you could say almost verbatim if asked cold. These are the concepts that show up underneath almost every data engineering interview, regardless of which specific stack (Fabric, Databricks, Snowflake, plain AWS) the role uses.

## Table of Contents

**A. Database & Storage Fundamentals**
1. OLTP vs. OLAP
2. ACID Properties
3. Normalization vs. Denormalization
4. Indexing
5. Primary/Foreign Keys & Referential Integrity

**B. Data Modeling**
6. Star Schema vs. Snowflake Schema
7. Fact Tables vs. Dimension Tables
8. Slowly Changing Dimensions (SCD)

**C. Data Movement Paradigms**
9. ETL vs. ELT
10. Batch vs. Streaming Processing
11. Data Pipelines & DAG-based Orchestration
12. Change Data Capture (CDC)
13. Idempotency

**D. Storage Architectures & Formats**
14. Data Warehouse vs. Data Lake vs. Lakehouse
15. Row-based vs. Columnar File Formats
16. Partitioning & Bucketing

**E. Distributed Systems & Processing**
17. Distributed Computing & MapReduce
18. Apache Spark Architecture
19. Data Skew & Shuffling
20. CAP Theorem
21. Horizontal vs. Vertical Scaling

**F. Streaming & Messaging**
22. Message Queues / Pub-Sub (Kafka model)
23. Delivery Semantics: At-Least-Once, At-Most-Once, Exactly-Once
24. Windowing (Tumbling, Hopping, Sliding)

**G. Data Quality & Governance**
25. Data Quality Dimensions
26. Data Lineage
27. Data Governance, Catalogs, and MDM (basics)

**H. SQL Fundamentals Worth Re-Sharpening**
28. Joins
29. Window Functions
30. Reading a Query Execution Plan

---

# A. Database & Storage Fundamentals

## 1. OLTP vs. OLAP

**What it is:** OLTP (Online Transaction Processing) systems run the operational, day-to-day transactions of a business — placing an order, updating a balance, inserting one row at a time, lots of small fast writes. OLAP (Online Analytical Processing) systems answer analytical questions over large volumes of historical data — "what were total sales by region last quarter" — favoring large scans and aggregations over single-row lookups.

**Why it matters (example):** An e-commerce order system is OLTP — every checkout is a fast, isolated transaction that must succeed or fail atomically. A nightly report computing revenue trends across 3 years of orders is OLAP — it scans millions of rows and aggregates, and would actively harm the OLTP system's performance if run directly against it. This is the entire reason data warehouses exist as a separate system from production databases.

**Trade-offs:**

| | OLTP | OLAP |
|---|---|---|
| Optimized for | Many small fast writes/reads | Few large scans/aggregations |
| Schema | Highly normalized (3NF) | Denormalized (star schema) |
| Typical engine | PostgreSQL, MySQL, SQL Server | Snowflake, BigQuery, Redshift, Synapse |
| Query pattern | `SELECT * WHERE id = ?` | `SELECT SUM(...) GROUP BY ...` |

**Diagram:**
```
[App] --writes--> [OLTP DB] --ETL/ELT--> [OLAP Warehouse] <--queries-- [BI tool]
```

**Interview-ready answer:** "OLTP systems handle the live, transactional side of the business — fast, small, row-level operations that need strong consistency. OLAP systems are built for analytical queries over historical data — large scans, aggregations, denormalized schemas. You almost never query OLTP directly for analytics because it would compete with production traffic; instead you move data into an OLAP-optimized warehouse via ETL/ELT."

---

## 2. ACID Properties

**What it is:** Atomicity, Consistency, Isolation, Durability — the guarantees a transactional database makes about how operations behave, especially under concurrency and failure.

**Why it matters (example):** A bank transfer needs to debit one account and credit another as a single atomic unit — if the credit fails after the debit succeeds, money disappears. Atomicity guarantees both happen or neither does. Durability guarantees that once the database says "committed," a power outage a millisecond later doesn't lose it.

**Trade-offs:** Strict ACID guarantees come at a cost — distributed systems that need ACID across nodes (e.g., distributed SQL databases) pay a coordination/latency cost that pure NoSQL/eventual-consistency systems avoid by relaxing these guarantees (often trading Consistency for Availability — see the CAP theorem, #20).

**Interview-ready answer:** "ACID is the contract a transactional database makes: Atomicity means a transaction is all-or-nothing, Consistency means it never leaves the database in an invalid state relative to its constraints, Isolation means concurrent transactions don't see each other's half-finished work, and Durability means a committed transaction survives a crash. It matters in data engineering whenever you're writing to an OLTP source or doing transactional upserts into a warehouse — MERGE statements and Delta Lake transactions, for example, lean on these same guarantees."

---

## 3. Normalization vs. Denormalization

**What it is:** Normalization organizes data to eliminate redundancy by splitting it into related tables (1NF, 2NF, 3NF rules) — every fact stored exactly once. Denormalization deliberately reintroduces redundancy by combining data into fewer, wider tables to make read queries faster, at the cost of update complexity and storage.

**Why it matters (example):** In a normalized OLTP schema, a customer's address lives in one `Customers` table and is referenced by ID everywhere — change it once, it's correct everywhere. In a denormalized OLAP fact table, you might duplicate the customer's region directly onto every order row, because joining to a `Customers` table for every one of 50 million order rows is exactly the kind of expensive join an analytical query wants to avoid.

**Trade-offs:**

| | Normalized | Denormalized |
|---|---|---|
| Storage | Minimal redundancy | More storage used |
| Write/update | Single source, easy to update | Update touches many duplicated rows |
| Read/query | Requires joins | Fast, fewer/no joins |
| Typical use | OLTP | OLAP |

**Interview-ready answer:** "Normalization removes redundancy by splitting data across related tables, which keeps writes clean and consistent but means reads need joins. Denormalization intentionally duplicates data into fewer, wider tables to make analytical reads fast, accepting the cost of update complexity and extra storage. OLTP systems lean normalized; OLAP/warehouse schemas lean denormalized — that's the whole reasoning behind a star schema."

---

## 4. Indexing

**What it is:** A separate data structure (commonly a B-tree or hash index) that lets the database find rows matching a condition without scanning every row — at the cost of extra storage and slower writes (the index has to be maintained on every insert/update).

**Why it matters (example):** `SELECT * FROM orders WHERE customer_id = 4521` on a 50-million-row table without an index on `customer_id` forces a full table scan. With an index, it's a near-instant lookup. But if you index every column "just in case," every `INSERT` now has to update every one of those indexes, and write throughput collapses.

**Trade-offs:** Indexes speed up reads on the indexed column(s) but slow down writes and cost storage — choosing what to index is a read/write trade-off, not a free win. Indexes also only help if the query actually uses them (e.g., a function applied to the column, like `WHERE UPPER(name) = ...`, often can't use a plain index on `name`).

**Interview-ready answer:** "An index lets the engine find matching rows without scanning the full table, dramatically speeding up selective reads — but every index adds write overhead and storage cost, since it has to be kept in sync on every insert, update, or delete. The skill isn't 'add an index,' it's knowing which columns are actually selective and queried often enough to justify that write-side cost."

---

## 5. Primary/Foreign Keys & Referential Integrity

**What it is:** A primary key uniquely identifies a row in a table; a foreign key references a primary key in another table, and referential integrity is the guarantee that a foreign key value always points to a row that actually exists.

**Why it matters (example):** An `Orders` table with `customer_id` as a foreign key to `Customers.customer_id` should never be allowed to contain an order pointing to a customer that doesn't exist — the database enforces this automatically if the constraint is declared, rather than relying on application code to remember to check.

**Trade-offs:** Enforcing referential integrity at the database level is safer but adds overhead on every write and can complicate operations like deleting a parent row (cascade delete vs. restrict). Many high-throughput analytical/warehouse tables deliberately *don't* enforce foreign key constraints at the engine level (most columnar warehouses don't even support enforced FKs) — the constraint becomes a documented/assumed relationship rather than a database-enforced one, and pipeline-level data quality checks pick up the slack.

**Interview-ready answer:** "A primary key uniquely identifies each row; a foreign key creates a relationship to another table's primary key, and referential integrity is the guarantee that relationship is never violated. In OLTP systems this is usually enforced by the database itself. In most analytical warehouses, foreign keys are often declared for documentation and join-optimizer hints but not strictly enforced at write time — data quality checks in the pipeline are what actually catch orphaned records."

---

# B. Data Modeling

## 6. Star Schema vs. Snowflake Schema

**What it is:** A star schema has one central fact table directly connected to several denormalized dimension tables (one join hop). A snowflake schema normalizes those dimensions further into sub-dimensions, creating a multi-hop join structure that looks like a snowflake when diagrammed.

**Diagram:**
```
Star:                          Snowflake:
   DimCustomer                    DimCustomer --- DimRegion
        \                              \
DimDate--FactSales--DimProduct   DimDate--FactSales--DimProduct--DimCategory
        /
   DimStore
```

**Why it matters (example):** `DimProduct` in a star schema might directly contain a `CategoryName` column (denormalized). In a snowflake schema, `CategoryName` would live in a separate `DimCategory` table that `DimProduct` joins to — saving storage if categories repeat a lot, but costing an extra join on every query that needs category.

**Trade-offs:** Star schemas are simpler and faster to query (fewer joins) — the standard default for BI tools and DirectLake/VertiPaq-style engines that are optimized for this exact shape. Snowflake schemas save storage and reduce update anomalies in the dimension data itself, at the cost of query complexity and slightly worse BI tool performance.

**Interview-ready answer:** "A star schema keeps dimensions denormalized and one join-hop from the fact table — simple, fast for BI tools. A snowflake schema normalizes dimensions further into related sub-tables, saving storage but adding join hops. In practice, most modern BI/semantic layers (including DirectLake-style engines) are specifically optimized for star schemas, so snowflaking is usually a deliberate trade you make only when dimension storage or update-anomaly concerns outweigh query simplicity."

---

## 7. Fact Tables vs. Dimension Tables

**What it is:** A fact table holds the measurable, numeric events of the business at a specific grain (e.g., one row per sale) plus foreign keys to dimensions. A dimension table holds the descriptive context around those facts (who, what, where, when) — typically wide, slowly changing, and much smaller in row count than the fact table.

**Why it matters (example):** `FactSales` (millions of rows: `date_key`, `product_key`, `customer_key`, `quantity`, `amount`) versus `DimProduct` (thousands of rows: `product_key`, `product_name`, `category`, `brand`). The fact table is what you sum/count; the dimension table is what you filter/group by.

**Interview-ready answer:** "Fact tables hold the numeric, measurable events at a defined grain — that grain statement, like 'one row per order line item,' is the single most important design decision in the table. Dimension tables hold the descriptive attributes you filter and group by. Getting the fact table's grain wrong — mixing two different grains in one fact table — is the single most common dimensional modeling mistake."

---

## 8. Slowly Changing Dimensions (SCD)

**What it is:** A pattern for handling dimension attributes that change over time — e.g., a customer moves cities, a product gets reassigned to a different category. SCD Type 0 means never update (fixed attribute). Type 1 overwrites the old value (no history kept). Type 2 inserts a new row with effective-date columns, preserving full history.

**Why it matters (example):** If a customer's region changes and you use Type 1, every historical sale now appears to have happened in their *new* region — which silently rewrites history and breaks "sales by region over time" reporting. Type 2 instead adds a new dimension row with a `valid_from`/`valid_to` date range, so historical facts still join to the region that was correct *at the time of that sale*.

**Trade-offs:** Type 1 is simple and cheap but destroys history. Type 2 preserves history accurately but adds complexity (surrogate keys, effective-dating logic, and the fact table must join on the dimension's surrogate key valid at the time of the event, not the natural key).

**Interview-ready answer:** "SCD Type 1 overwrites a changed dimension value with no history — simple, but it rewrites the past. SCD Type 2 inserts a new versioned row with effective-date ranges, preserving exactly what was true at any point in history, which is what you need whenever 'what was true at the time' actually matters for reporting — like sales by sales-rep territory when territories get reassigned. The cost is added complexity: you need a surrogate key strategy and the fact table has to reference the dimension version that was active when the fact occurred."

---

# C. Data Movement Paradigms

## 9. ETL vs. ELT

**What it is:** ETL (Extract, Transform, Load) transforms data *before* loading it into the target system — transformation happens in a separate processing layer. ELT (Extract, Load, Transform) loads raw data into the target first, then transforms it using the target system's own compute (e.g., a warehouse's SQL engine, or Spark on a lakehouse).

**Diagram:**
```
ETL:  Source --> [Transform engine] --> Target (clean data only)
ELT:  Source --> Target (raw landing) --> [Transform inside Target] --> Target (clean tables)
```

**Why it matters (example):** Classic ETL tools (Informatica, SSIS) transformed data on a dedicated server before it ever touched the warehouse — necessary when warehouses had limited, expensive compute. Modern cloud warehouses (Snowflake, BigQuery) have cheap, elastic compute, so ELT became dominant: dump raw JSON/CSV into the warehouse or lake first (Bronze layer), then transform with SQL or Spark using the warehouse/lake's own scalable compute (Silver/Gold layers) — this is exactly the Medallion architecture pattern.

**Trade-offs:** ELT gives you the raw data preserved (useful for reprocessing/debugging/changing transformation logic later without re-extracting from source) and leverages cheap modern warehouse compute, but means raw, possibly messy/sensitive data lands in the target before it's cleaned. ETL keeps unclean/sensitive raw data out of the target entirely, at the cost of needing separate transformation infrastructure and losing easy access to the raw form if requirements change later.

**Interview-ready answer:** "ETL transforms data before it lands in the target system, using a separate processing layer. ELT loads raw data in first and transforms it using the target's own compute — which is why ELT became dominant once cloud warehouses got cheap, elastic compute. The big practical advantage of ELT is that you keep the raw data around — Bronze layer in a Medallion architecture — so if your transformation logic was wrong, you reprocess from the preserved raw data instead of re-extracting from the source system."

---

## 10. Batch vs. Streaming Processing

**What it is:** Batch processing runs on a defined chunk of accumulated data on a schedule (hourly, nightly). Streaming processing handles each event (or micro-batch of events) as it arrives, continuously, with much lower end-to-end latency.

**Why it matters (example):** A nightly sales report is naturally batch — there's no need for sub-second freshness. Fraud detection on credit card swipes needs streaming — waiting until tonight's batch job to flag a fraudulent transaction is useless; the decision has to happen within milliseconds of the swipe.

**Trade-offs:** Batch is simpler to build, test, and reason about (a batch job either ran or didn't; you can re-run it idempotently against a known input). Streaming introduces real complexity: out-of-order events, late arrivals, windowing semantics, and exactly-once delivery concerns that simply don't exist in batch. The honest engineering default is "use batch unless you have a specific, articulable latency requirement that justifies streaming's added complexity" — not "streaming is more modern, default to it."

**Interview-ready answer:** "Batch processes accumulated data on a schedule; streaming processes events continuously as they arrive, trading implementation complexity for much lower latency. The decision should be driven by an actual business latency requirement — fraud detection or real-time alerting genuinely needs streaming, but a lot of 'we should make this real-time' requests don't actually have a latency requirement that justifies the added complexity of out-of-order handling, windowing, and delivery semantics that streaming brings."

---

## 11. Data Pipelines & DAG-based Orchestration

**What it is:** A data pipeline is a sequence of steps (extract, transform, load, validate) that move and reshape data. Orchestration tools (Airflow, Fabric Pipelines, Dagster) represent these steps as a DAG (Directed Acyclic Graph) — tasks with explicit dependencies, no cycles, so the orchestrator knows what can run in parallel and what must wait.

**Diagram:**
```
extract_orders --\
                  +--> transform_join --> load_warehouse --> run_quality_checks
extract_customers-/
```

**Why it matters (example):** `transform_join` can't run until both `extract_orders` and `extract_customers` finish — the DAG encodes that dependency explicitly so the orchestrator runs the two extracts in parallel and only starts the join once both succeed, rather than a human manually sequencing scripts.

**Interview-ready answer:** "A DAG models a pipeline's tasks and their dependencies explicitly — no cycles, so there's always a valid execution order, and independent branches can run in parallel. Orchestrators like Airflow use this to handle retries, scheduling, and failure isolation per task rather than treating the whole pipeline as one monolithic script that either fully succeeds or fully fails."

---

## 12. Change Data Capture (CDC)

**What it is:** A technique for capturing row-level inserts/updates/deletes from a source database as they happen (typically by reading the database's transaction/write-ahead log) and streaming them to downstream consumers, instead of repeatedly querying the whole table to find what changed.

**Why it matters (example):** Re-querying a 500-million-row orders table every 15 minutes to find "what changed" (`WHERE updated_at > last_run`) is expensive and can miss hard deletes entirely (a deleted row leaves no `updated_at` to filter on). CDC reads the database's transaction log directly, capturing every insert/update/delete — including deletes — as a discrete event, with minimal load on the source system.

**Trade-offs:** CDC is more architecturally complex to set up (often needs a dedicated tool like Debezium, or a managed CDC feature like Fabric Mirroring) but captures changes completely and with much lower latency and source-system load than periodic full/incremental re-querying.

**Interview-ready answer:** "CDC captures row-level changes — inserts, updates, and deletes — directly from a source database's transaction log as they happen, rather than repeatedly querying the table to infer what changed. It's the right answer whenever a naive 'query for rows newer than my last watermark' approach would miss hard deletes or put unacceptable repeated load on a production source system."

---

## 13. Idempotency

**What it is:** An operation is idempotent if running it multiple times produces the same result as running it once. In data pipelines, this means re-running a failed or retried job doesn't create duplicate data or corrupt state.

**Why it matters (example):** A pipeline that does a blind `INSERT` for every row in a file will create duplicates if the job is retried after a partial failure (some rows already inserted, then it reruns from the start). A pipeline that does a `MERGE`/upsert keyed on a natural business key, instead, can be safely re-run any number of times — matching rows just update to the same values, no duplicates appear.

**Interview-ready answer:** "Idempotency means running a pipeline twice on the same input produces the same end state as running it once — which matters constantly in data engineering because retries are inevitable, whether from a transient failure, an orchestrator re-triggering a task, or a manual backfill. The standard mechanism is keying writes on a natural business key with MERGE/upsert logic rather than blind inserts, so reprocessing is always safe."

---

# D. Storage Architectures & Formats

## 14. Data Warehouse vs. Data Lake vs. Lakehouse

**What it is:** A data warehouse stores structured, schema-on-write data optimized for fast SQL analytics (Snowflake, Redshift, Synapse). A data lake stores raw data of any structure (structured, semi-structured, unstructured) cheaply at scale, with schema applied at read time (schema-on-read). A lakehouse combines lake-style cheap, flexible storage with warehouse-style transactional guarantees and performance, typically via an open table format like Delta Lake or Iceberg.

**Diagram:**
```
Warehouse:  structured data --schema-on-write--> fast SQL analytics
Lake:       any data        --schema-on-read -->  flexible, cheap, but slower/less governed
Lakehouse:  any data        --Delta/Iceberg table format--> ACID transactions + fast analytics on lake storage
```

**Why it matters (example):** Before lakehouses, a company often ran both a lake (for raw/ML/unstructured data) and a separate warehouse (for governed BI), duplicating data and reconciliation effort between the two. A lakehouse (Fabric's Lakehouse item, Databricks) lets one copy of the data, stored as Delta files, serve both the flexible Spark/ML workloads and fast, ACID-compliant SQL analytics — which is the architectural bet behind Fabric's entire OneLake model.

**Interview-ready answer:** "A warehouse is schema-on-write, structured, and optimized for fast governed SQL analytics. A data lake is schema-on-read, stores any data type cheaply at scale, but historically lacked the transactional guarantees and performance warehouses had. A lakehouse closes that gap by adding an open, ACID-transactional table format like Delta or Iceberg on top of lake storage, so you get warehouse-grade reliability and performance without giving up the lake's flexibility and cost profile — which is why most new platforms are converging on the lakehouse pattern rather than running both separately."

---

## 15. Row-based vs. Columnar File Formats

**What it is:** Row-based formats (CSV, JSON, Avro) store each record's fields together, contiguous on disk. Columnar formats (Parquet, ORC) store each column's values together across all records, contiguous on disk.

**Diagram:**
```
Row-based:        Row1: [id, name, amount] | Row2: [id, name, amount] | ...
Columnar:          id column: [1,2,3,...] | name column: [a,b,c,...] | amount column: [10,20,30,...]
```

**Why it matters (example):** A query that only needs `SUM(amount)` across 10 million rows with 50 columns, on a columnar format, only has to read the `amount` column's bytes off disk. On a row-based format, it has to read every column of every row just to extract the one field it needs — far more I/O for the exact same analytical query.

**Trade-offs:** Columnar formats are dramatically better for analytical (OLAP-style) read patterns and compress better (similar values stored together compress more efficiently), but are worse for writing/updating single rows one at a time (an OLTP-style pattern), since a single-row write touches every column's separate storage. Row-based formats are the natural fit for OLTP and for simple interchange (CSV/JSON are human-readable and universally supported, at the cost of size and lack of embedded schema/type information compared to Avro/Parquet).

**Interview-ready answer:** "Row-based formats store each record's fields together, which suits OLTP-style single-row reads/writes and simple interchange. Columnar formats store each column together across all rows, which means an analytical query only reads the columns it actually needs — far less I/O — and compresses better since similar values sit next to each other. That's exactly why Parquet became the default storage format for lakes and lakehouses: almost all analytical queries are columnar-access patterns."

---

## 16. Partitioning & Bucketing

**What it is:** Partitioning physically splits a table's files into separate folders based on a column's value (e.g., `/year=2026/month=06/`), so queries filtering on that column can skip entire folders without reading them. Bucketing splits data into a fixed number of files based on a hash of a column, useful for optimizing joins on that column even when it's not a natural partition key (too high-cardinality to partition by directly).

**Why it matters (example):** Partitioning a 500-million-row events table by `event_date` means a query filtering `WHERE event_date = '2026-06-01'` only reads that one day's folder, not the entire table's history. But partitioning by a high-cardinality column like `user_id` (millions of distinct values) would create millions of tiny folders/files — the "small file problem" — which is worse than not partitioning at all.

**Trade-offs:** Good partition column choice = low-to-moderate cardinality, frequently used in query filters, aligned with how data naturally arrives (date is the overwhelmingly common choice). Over-partitioning (too many small partitions) hurts more than it helps via file/metadata overhead — this is the most common mistake junior engineers make with partitioning.

**Interview-ready answer:** "Partitioning splits files into folders by a column's value so queries can skip irrelevant folders entirely — date is the classic partition column because it's low cardinality and almost every analytical query filters by some date range. The failure mode to watch for is over-partitioning on a high-cardinality column, which creates a 'small file problem' — too many tiny files with proportionally large metadata overhead — that hurts performance more than having no partitioning at all."

---

# E. Distributed Systems & Processing

## 17. Distributed Computing & MapReduce

**What it is:** Instead of processing a huge dataset on one machine, distributed computing splits the data and the work across many machines that process their slice in parallel, then combine results. MapReduce is the foundational pattern: a "Map" phase processes each piece of data independently (e.g., count words in this chunk), and a "Reduce" phase aggregates the mapped results together (sum the counts across all chunks).

**Why it matters (example):** Counting word frequency across a 10TB text corpus on one machine could take days and might not even fit in memory. Splitting it across 100 machines, each mapping its own 100GB slice and emitting partial counts, then reducing (summing) those partial counts together, turns an intractable single-machine job into a parallel one that finishes in minutes.

**Interview-ready answer:** "MapReduce splits a large computation into a Map phase, where each chunk of data is processed independently and in parallel, and a Reduce phase, where those partial results get combined into a final answer. It's the conceptual foundation underneath Spark, Hadoop, and most distributed data processing — even though modern engines like Spark generalize well beyond literal map/reduce into arbitrary DAGs of transformations."

---

## 18. Apache Spark Architecture

**What it is:** A Driver process plans the work and coordinates Executor processes, which actually run the computation in parallel across a cluster's worker nodes. Spark builds a logical plan of transformations (lazy evaluation — nothing actually runs until an action like `.collect()`, `.write()`, or `.count()` is called), then optimizes and executes that plan as a DAG of stages and tasks distributed across executors.

**Diagram:**
```
            Driver (plans + coordinates)
           /        |        \
   Executor 1   Executor 2   Executor 3
   (tasks)      (tasks)      (tasks)
```

**Why it matters (example):** Writing `df.filter(...).select(...).groupBy(...)` in PySpark doesn't execute anything immediately — Spark builds a logical plan. Only when you call `.write()` or `.collect()` does Spark's optimizer (Catalyst) turn that plan into an actual physical execution DAG and run it across executors. This is why a stack trace error often points to the *action* line rather than the *actual* problematic transformation several lines earlier — the whole plan executes at once when triggered.

**Trade-offs/sharp edge:** `.collect()` pulls all results back to the single Driver's memory — fine for a small aggregated result, dangerous (OOM) for an unfiltered large dataset, since the Driver is one machine, not the distributed cluster.

**Interview-ready answer:** "Spark has a Driver that builds and coordinates a logical execution plan, and Executors that run the actual distributed work. Spark is lazily evaluated — transformations like filter and select just build up a plan, and nothing actually runs until an action like collect, write, or count triggers execution. That laziness is powerful for optimization but it's also why debugging means tracing back through the whole chain of transformations, not just looking at the line where the action — and therefore the failure — appears."

---

## 19. Data Skew & Shuffling

**What it is:** A shuffle is when Spark (or any distributed engine) has to redistribute data across the network between stages — typically required by operations like `groupBy`, `join`, or `repartition`, because related rows need to end up on the same machine to be processed together. Data skew is when that redistributed data isn't evenly spread — some partitions end up far larger than others, so a few tasks take dramatically longer than the rest, while most executors sit idle waiting.

**Why it matters (example):** Joining a table where 80% of rows share one `customer_id` value (e.g., a "guest checkout" placeholder ID) means the task handling that one key has to process 80% of the data alone — the job's total runtime becomes bottlenecked on that single skewed task, no matter how many executors you throw at it.

**Trade-offs/mitigation:** Salting the skewed key (artificially splitting it into several sub-keys to spread the load), broadcasting a small table in a join instead of shuffling both sides, or pre-aggregating before the shuffle are the standard mitigations — simply adding more executors does nothing for skew, since the bottleneck is one oversized task, not insufficient total capacity.

**Interview-ready answer:** "A shuffle redistributes data across the network so related rows land together for an operation like a join or group-by. Data skew happens when that redistribution is uneven — one key has far more rows than others — so one task becomes the bottleneck while the rest of the cluster idles. The fix isn't more executors, since the problem is one oversized task, not total capacity — it's techniques like salting the skewed key or broadcasting a small table to avoid the shuffle on the large side entirely."

---

## 20. CAP Theorem

**What it is:** In a distributed system, you can't simultaneously guarantee all three of Consistency (every read sees the latest write), Availability (every request gets a response), and Partition tolerance (the system keeps working despite network failures between nodes) — when a network partition happens, you must choose between consistency and availability.

**Why it matters (example):** During a network split between two data centers, a CP system (e.g., many traditional distributed SQL databases) will refuse to serve requests on the minority side to avoid returning stale/inconsistent data — sacrificing availability. An AP system (e.g., Cassandra, DynamoDB in its default mode) keeps serving requests on both sides, but the two sides may temporarily disagree on the current value — sacrificing strict consistency for eventual consistency instead.

**Interview-ready answer:** "CAP theorem says a distributed system facing a network partition has to choose between consistency and availability — it can't have both at that moment, though partition tolerance itself is generally assumed as a given for any real distributed system. It mostly comes up in data engineering when choosing a distributed database — a system like Cassandra or DynamoDB chooses availability with eventual consistency, while a strongly consistent distributed database sacrifices availability during a partition to guarantee every read reflects the latest write."

---

## 21. Horizontal vs. Vertical Scaling

**What it is:** Vertical scaling (scaling up) means adding more resources (CPU, RAM) to a single machine. Horizontal scaling (scaling out) means adding more machines and distributing the work across them.

**Why it matters (example):** A single PostgreSQL instance can be vertically scaled by moving to a bigger VM — simple, but there's a hard ceiling (the biggest VM available) and a single point of failure. A Spark cluster scales horizontally — adding more worker nodes — which has a much higher practical ceiling and better fault tolerance, but requires the workload to actually be parallelizable across machines, which not every workload is.

**Interview-ready answer:** "Vertical scaling means making one machine bigger; horizontal scaling means adding more machines and distributing the work. Distributed engines like Spark are built around horizontal scaling, which has a far higher ceiling and better fault tolerance than vertical scaling alone, but it only pays off if the workload can actually be parallelized — some workloads (like a single deeply sequential transformation with no natural split points) don't benefit from more machines no matter how you architect it."

---

# F. Streaming & Messaging

## 22. Message Queues / Pub-Sub (Kafka Model)

**What it is:** A message queue/pub-sub system decouples producers (who emit events) from consumers (who process them), buffering events durably in between. Kafka's specific model: producers write events to a Topic, which is split into Partitions for parallelism, and Consumer Groups read from those partitions, with each partition consumed by only one consumer within a given group at a time (this is what guarantees ordering within a partition).

**Diagram:**
```
Producer --> Topic [Partition 0, Partition 1, Partition 2] --> Consumer Group
                                                                  (Consumer A reads P0)
                                                                  (Consumer B reads P1, P2)
```

**Why it matters (example):** If your order-processing service writes directly to your inventory service's database, the two are tightly coupled — inventory's downtime breaks order processing. Putting Kafka in between means the order service just publishes an "order placed" event and moves on; inventory consumes it whenever it's ready, and a temporary inventory outage doesn't lose data — it just builds up a backlog the consumer catches up on once it recovers.

**Interview-ready answer:** "A pub-sub system like Kafka decouples producers from consumers — producers write events to a topic without knowing or caring who's consuming them, and the broker durably buffers those events. Kafka specifically splits a topic into partitions for parallelism, and within a consumer group, each partition is owned by exactly one consumer at a time, which is what gives you ordering guarantees within a partition while still allowing horizontal scale-out across partitions."

---

## 23. Delivery Semantics: At-Least-Once, At-Most-Once, Exactly-Once

**What it is:** These describe the guarantee a messaging/streaming system makes about how many times a given message will be processed. At-most-once: a message might be lost but never processed twice. At-least-once: a message is never lost but might be processed more than once (e.g., on retry after a failure). Exactly-once: a message is processed precisely one time — the hardest and most expensive guarantee to actually achieve end-to-end.

**Why it matters (example):** If a consumer crashes after processing a message but before acknowledging it, an at-least-once system will redeliver that message — meaning your downstream processing logic must be idempotent (#13) to avoid double-counting, since "at-least-once delivery + idempotent processing" is how most real systems achieve effectively-exactly-once outcomes without paying for true exactly-once machinery everywhere.

**Interview-ready answer:** "At-most-once can lose messages but never duplicates them; at-least-once never loses messages but can deliver duplicates, typically from retries after a failure; exactly-once guarantees precisely one delivery, which is genuinely hard and expensive to achieve end-to-end across distributed systems. In practice, most production systems choose at-least-once delivery paired with idempotent consumer logic — which gets you the practical outcome of exactly-once processing without needing true exactly-once semantics baked into the transport layer itself."

---

## 24. Windowing (Tumbling, Hopping, Sliding)

**What it is:** Since streaming data never "ends," aggregations need a window of time to operate over. A tumbling window is fixed-size and non-overlapping (every event belongs to exactly one window). A hopping window is fixed-size but overlapping, advancing by a smaller "hop" interval than the window size (an event can belong to multiple windows). A sliding window continuously re-evaluates as new events arrive, rather than at fixed intervals.

**Diagram:**
```
Tumbling (5-min, 5-min hop):  [0-5][5-10][10-15]            -- no overlap
Hopping  (5-min, 1-min hop):  [0-5][1-6][2-7][3-8]...        -- overlapping
```

**Why it matters (example):** "Average order value per 5-minute block, no overlap" is a tumbling window — clean, non-overlapping reporting buckets. "A 5-minute moving average updated every minute, for a smoother dashboard trend line" is hopping. "Alert the instant a 30-second rolling average crosses a threshold" is closer to sliding/continuous evaluation, usually implemented via a continuously-maintained materialized aggregate rather than a single ad-hoc recompute.

**Interview-ready answer:** "Tumbling windows are fixed and non-overlapping — every event belongs to exactly one window, which is the right choice for clean reporting buckets. Hopping windows overlap — useful for a smoothed moving-average view that updates more often than the window itself. Sliding/continuous windows re-evaluate as events arrive rather than on a fixed schedule, and in practice that's usually implemented as a continuously maintained aggregate rather than literally recomputing from scratch on every single event."

---

# G. Data Quality & Governance

## 25. Data Quality Dimensions

**What it is:** The standard dimensions used to evaluate whether data is "good": completeness (are required values present), accuracy (do values correctly reflect reality), consistency (does the same fact agree across systems/tables), timeliness (is the data fresh enough to be useful), uniqueness (no unintended duplicates), and validity (does data conform to expected format/range/type).

**Why it matters (example):** A customer table that's 100% complete (no nulls) can still be wrong if half the email addresses are typo'd (an accuracy problem, not a completeness one) — the dimensions are genuinely independent and a pipeline can pass a completeness check while failing badly on accuracy.

**Interview-ready answer:** "Completeness, accuracy, consistency, timeliness, uniqueness, and validity are the standard dimensions for evaluating data quality, and the important interview point is that they're independent — a dataset can have zero nulls and still be inaccurate, or be perfectly accurate at the moment of capture but stale and therefore low-quality by the time it's used. A real data quality program checks multiple dimensions, not just 'are there nulls.'"

---

## 26. Data Lineage

**What it is:** The traceable record of where a piece of data came from, what transformations it went through, and where it ends up — essentially a map of a dataset's life from source to consumption.

**Why it matters (example):** When a finance dashboard's revenue number looks wrong, lineage lets you trace backward — which source tables fed this number, which transformation logic touched it, when it last ran — rather than manually re-deriving the entire pipeline's logic from scratch under pressure during an incident.

**Interview-ready answer:** "Data lineage is the traceable map of where data came from and what happened to it on the way to where it's used. It matters most during incident response — when a number looks wrong, lineage is how you trace backward to the actual source of the problem instead of guessing — and for impact analysis before making a change, since lineage tells you exactly which downstream reports or models a given upstream table change would affect."

---

## 27. Data Governance, Catalogs, and MDM (Basics)

**What it is:** Data governance is the overall set of policies, roles, and processes for managing data as an organizational asset (who owns what, who can access what, what quality bar is required). A data catalog is the searchable inventory of what data exists, where, and what it means (often paired with lineage and quality metadata). Master Data Management (MDM) specifically tackles maintaining one trusted, de-duplicated version of core business entities (a single "Customer" record, not five conflicting ones across five systems).

**Why it matters (example):** Without MDM, a "customer" might exist as three slightly different records across CRM, billing, and support systems (different spellings, different IDs) — any cross-system analysis silently undercounts or double-counts that customer until MDM resolves them into one master record with a mapping back to each source system's local ID.

**Interview-ready answer:** "Governance is the policy and ownership layer for data as an asset; a catalog is the searchable inventory of what data exists and what it means; MDM specifically resolves the same real-world entity appearing differently across multiple systems into one trusted master record. They're related but distinct — you can have a catalog with zero governance enforcement, and you can have strong governance without ever having tackled MDM, if your organization hasn't yet hit the 'same customer, three conflicting records' problem at scale."

---

# H. SQL Fundamentals Worth Re-Sharpening

## 28. Joins

**What it is:** A way of combining rows from two or more tables based on a related column. Inner join returns only matching rows in both tables. Left (outer) join returns all rows from the left table plus matches from the right (nulls where no match). Full outer join returns all rows from both, matched where possible. Cross join returns every combination of rows from both tables (a Cartesian product) — generally a mistake unless deliberately intended.

**Diagram:**
```
Inner:  A ∩ B            Left:  all of A + matched B        Full: all of A + all of B
```

**Why it matters (example):** Using an inner join when you actually need a left join silently drops rows — e.g., joining `Orders` to `Returns` with an inner join means orders with no return simply vanish from the result, which is exactly wrong if the goal was "all orders, with return info where it exists."

**Interview-ready answer:** "Inner join keeps only rows that match in both tables; left join keeps everything from the left table and fills in nulls where there's no match on the right; full outer keeps everything from both sides. The most common real-world bug is reaching for an inner join out of habit when a left join was actually needed, silently dropping rows that had no match — which is a quiet correctness bug, not something that throws an error."

---

## 29. Window Functions

**What it is:** SQL functions that compute a value across a set of rows related to the current row (a "window," defined by `PARTITION BY` and `ORDER BY`) without collapsing those rows into a single aggregated result the way `GROUP BY` does. Common examples: `ROW_NUMBER()`, `RANK()`, running totals via `SUM(...) OVER (...)`, and `LAG()`/`LEAD()` for comparing a row to a previous/next row.

**Why it matters (example):** "Each customer's order ranked by order date" needs `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)` — a plain `GROUP BY` can't produce this because it would collapse each customer down to one row, losing the per-order detail you still need alongside the rank.

**Interview-ready answer:** "Window functions compute a value across a related set of rows — defined by PARTITION BY and ORDER BY — without collapsing those rows the way GROUP BY does. They're the right tool whenever you need both row-level detail and an aggregate-like calculation in the same result, like a running total, a rank within a group, or comparing a row to the previous one with LAG."

---

## 30. Reading a Query Execution Plan

**What it is:** The step-by-step physical strategy a database engine actually uses to execute a query — which indexes (if any) it uses, what join algorithm it picks (nested loop, hash join, merge join), and in what order it processes tables — viewable via `EXPLAIN` (or `EXPLAIN ANALYZE` for actual runtime stats, not just the estimated plan).

**Why it matters (example):** A query that "should" be fast because there's an index on the filtered column might still show a full table scan in its execution plan — often because a function was applied to the column (`WHERE YEAR(order_date) = 2026` instead of a direct range comparison), which prevents the optimizer from using the index at all. You can't know this without actually reading the plan; guessing from the SQL text alone isn't reliable.

**Interview-ready answer:** "An execution plan shows the actual physical strategy the engine chose — which join algorithm, whether an index was used, what order tables were processed in. The habit worth having is: don't assume a query is using an index just because one exists on the filtered column — always check the actual plan, since something as simple as wrapping the column in a function can silently disable index usage and force a full scan."

---

## Closing note for interview use

If asked any of these cold, the strongest answers follow the same shape every time: name the concept precisely, give one concrete example (not an abstract definition only), and name the trade-off or the failure mode it's solving for. Interviewers at the senior/architect level are listening for that trade-off framing far more than for textbook-perfect definitions — it's the signal that you've actually hit the failure mode in production, not just read about it.
