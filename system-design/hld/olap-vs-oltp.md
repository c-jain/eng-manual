---
Status: 🌳 Evergreen
Created: 2026-06-19
Last Updated: 2026-06-19
---

# Data Warehousing & OLAP vs OLTP

## Table of Contents

1. [The Core Problem](#the-core-problem)
2. [OLTP — Transactional Systems](#oltp--transactional-systems)
3. [OLAP — Analytical Systems](#olap--analytical-systems)
4. [OLTP vs OLAP At A Glance](#oltp-vs-olap-at-a-glance)
5. [What Is a Data Warehouse?](#what-is-a-data-warehouse)
6. [Data Warehouse Architecture](#data-warehouse-architecture)
7. [ETL vs ELT](#etl-vs-elt)
8. [Columnar Storage](#columnar-storage)
9. [Schema Design: Star and Snowflake](#schema-design-star-and-snowflake)
10. [OLAP Cube Operations](#olap-cube-operations)
11. [Data Lake vs Data Warehouse vs Data Lakehouse](#data-lake-vs-data-warehouse-vs-data-lakehouse)
12. [HTAP: Hybrid Systems](#htap-hybrid-systems)
13. [Partitioning and Materialized Views](#partitioning-and-materialized-views)
14. [Go Code: ETL Pipeline and Star Schema](#go-code-etl-pipeline-and-star-schema)
15. [Trade-Off Tables](#trade-off-tables)
16. [How to Remember This](#how-to-remember-this)
17. [Interview Cheat Sheet](#interview-cheat-sheet)
18. [References](#references)

---

## The Core Problem

Your OLTP database (Postgres, MySQL) handles millions of user-facing writes per second — orders, payments, inventory updates. Now the analytics team asks:

> *"What was revenue by product category and region, over the last 6 months, segmented by customer acquisition channel?"*

That query needs to scan hundreds of millions of rows, perform multiple GROUP BYs, and hold large aggregations in memory — potentially for minutes. Running it on your production DB would:

- Saturate read I/O and spike latency for real users
- Compete with time-critical writes
- Lock rows or degrade query plans
- Probably timeout before returning results

This gap — between systems optimised for *recording* events and systems optimised for *analysing* them — is what OLAP and Data Warehousing exist to fill.

---

## OLTP — Transactional Systems

### What It Is

Online Transaction Processing. Systems that handle the current, ongoing operations of a business. Every `INSERT`, `UPDATE`, and `DELETE` that keeps your product alive.

### Why It Is Named That

- **Online** = interactive and real-time, not batch
- **Transaction** = ACID-compliant unit of work (the bank transfer, the order placement)
- **Processing** = executing those units

### Characteristics

- High-volume, short-lived queries (microseconds to low milliseconds)
- Normalised schema (3NF) — minimises redundancy, fast point lookups
- **Row-oriented storage** — full row needed for a write or point-read
- Optimised for write throughput and high concurrency
- Indexes on primary keys and foreign keys
- Many concurrent short-lived connections
- Data volume per query: small (one or a few rows)

### Problems It Brings

- Normalised schema → many JOINs → terrible for aggregations
- Row storage → scanning one column forces reading all columns
- Analytics queries compete with user traffic and degrade SLOs
- Historical data bloats the operational DB and slows everything down

---

## OLAP — Analytical Systems

### What It Is

Online Analytical Processing. Systems designed to answer complex, multi-dimensional questions about *historical* data.

### Why It Is Named That

- **Online** = interactive, not pure batch — you can drill into data interactively
- **Analytical** = aggregations, trends, comparisons across enormous row counts
- **Processing** = executing large-scale reads efficiently

The term was coined by E.F. Codd (inventor of the relational model) in 1993 to formally distinguish analytical systems from OLTP.

### Characteristics

- Low volume of long, complex queries (seconds to minutes acceptable)
- **Denormalised schema** (Star or Snowflake) — fewer JOINs at query time
- **Column-oriented storage** — read only the columns the query needs
- Optimised for read throughput on billions of rows
- Fewer concurrent users (analysts, scheduled dashboards)
- Aggregations dominate: GROUP BY, SUM, AVG, COUNT DISTINCT, WINDOW functions
- Compression is highly effective — same data type per column

### Problems It Brings

- **Data lag** — always some delay from production to warehouse (minutes to hours)
- **Schema rigidity** — columnar formats make adding/removing columns expensive
- **Stale data** — not suitable for use cases requiring sub-second freshness
- **Operational complexity** — pipelines, transformations, freshness SLOs to maintain

---

## OLTP vs OLAP At A Glance

| Dimension | OLTP | OLAP |
|---|---|---|
| Purpose | Record operations | Analyse operations |
| Primary operation | INSERT / UPDATE / DELETE | Aggregate SELECT |
| Schema | Normalised (3NF) | Denormalised (Star / Snowflake) |
| Storage layout | Row-oriented | Column-oriented |
| Query latency | Sub-ms to low ms | Seconds to minutes |
| Concurrent users | Thousands | Tens to hundreds |
| Data age | Current | Historical |
| Optimised for | Write throughput | Read throughput |
| Data scanned per query | Small (rows by key) | Huge (full column scans) |
| Example systems | Postgres, MySQL, Oracle | BigQuery, Snowflake, Redshift, ClickHouse |

---

## What Is a Data Warehouse?

A **centralised repository** that aggregates data from multiple operational sources, cleans and transforms it, and makes it available for analytical queries.

The *warehouse* metaphor: a physical warehouse receives goods from many factories (sources), organises them systematically, and lets you retrieve them efficiently. The "goods" here are business facts.

### Four Key Properties (Inmon's Definition)

| Property | Meaning |
|---|---|
| Subject-oriented | Organised around domains (Sales, Finance, Users), not around operational systems |
| Integrated | Consistent naming, formats, and types across all sources |
| Non-volatile | Data is loaded and preserved; not updated in-place like OLTP |
| Time-variant | Full history is retained; you can query "state of the world at time T" |

---

## Data Warehouse Architecture

```
[Sources]              [Ingestion]              [Storage]             [Consumption]

OLTP DB  ──┐
Events   ──┼──► Staging ──► ETL / ELT ──► Data Warehouse ──► BI Tools
SaaS API ──┘    (raw)        Layer          (modelled)         Dashboards
                                                 │             Ad-hoc SQL
                                            Data Marts         Data Science
                                           (per domain)
```

**Staging Area:** Raw, unmodified data lands here first. Acts as a buffer and immutable audit log. You can always re-transform from staging if your transformation logic changes.

**Data Marts:** Subsets of the warehouse scoped to a business unit — Finance Mart, Marketing Mart. Not necessarily separate physical systems; often schema-level partitions or views. Provide faster queries and tighter access control per team.

---

## ETL vs ELT

Two philosophies for getting data into the warehouse.

### Scenario A — ETL (Extract, Transform, Load): Traditional

```
Source ──► Extract ──► Transform ──► Load ──► Warehouse
                        (external
                         compute)
```

Transformation happens *outside* the warehouse, in a dedicated compute layer (Spark, Hadoop, custom jobs). Only clean data is loaded.

Good when: storage was expensive (don't store junk), privacy/compliance requires filtering before storage, or transformation logic is too complex for SQL.

### Scenario B — ELT (Extract, Load, Transform): Modern

```
Source ──► Extract ──► Load ──► Warehouse ──► Transform
                                              (in-warehouse
                                               SQL via dbt)
```

Raw data lands in the warehouse first. Transformation uses the warehouse's own massive compute (BigQuery slots, Snowflake virtual warehouses). Tools like **dbt** own the transformation layer.

Good when: storage is cheap (S3/GCS-backed warehouses), the warehouse is powerful enough, and you want flexibility to re-transform raw data as business logic evolves.

**ELT is the dominant pattern today.** Storing raw data is cheap; losing it is expensive.

### ETL vs ELT Trade-Off

| Dimension | ETL | ELT |
|---|---|---|
| Where transform runs | External compute | Inside the warehouse |
| Data stored | Cleaned only | Raw + cleaned |
| Flexibility | Hard to re-transform | Re-run transform any time |
| Privacy / compliance | Easier (filter before storage) | Harder (raw data in warehouse) |
| Operational cost | External infra to maintain | Warehouse bill goes up |
| Modern tooling | Airflow + Spark | dbt + Airflow / Dagster |

---

## Columnar Storage

The most important internal concept for interviews.

### Row-Oriented Storage (OLTP)

Data is stored row by row. Reading any value requires reading the entire row.

```
┌────────────────────────────────────────────────────┐
│ Row 1: [id=1 │ name=Alice │ age=30 │ revenue=100] │
│ Row 2: [id=2 │ name=Bob   │ age=25 │ revenue=200] │
│ Row 3: [id=3 │ name=Carol │ age=28 │ revenue=300] │
└────────────────────────────────────────────────────┘
  SELECT SUM(revenue) → must read ALL columns for ALL rows
```

### Column-Oriented Storage (OLAP)

Data is stored column by column. A query touching 3 of 100 columns reads 3% of the data.

```
┌──────┐  ┌─────────┐  ┌───────┐  ┌───────────┐
│  id  │  │  name   │  │  age  │  │  revenue  │
├──────┤  ├─────────┤  ├───────┤  ├───────────┤
│  1   │  │  Alice  │  │  30   │  │    100    │
│  2   │  │  Bob    │  │  25   │  │    200    │
│  3   │  │  Carol  │  │  28   │  │    300    │
└──────┘  └─────────┘  └───────┘  └───────────┘
  SELECT SUM(revenue) → read ONLY the revenue column
```

### Why Columnar Storage Is Better for Analytics

**I/O reduction:** A 100-column table where your query touches 3 columns = 97% less data read from disk.

**Compression:** Same data type per column compresses far better. Techniques:
- *Run-length encoding*: "NY, NY, NY, NY, NY" → "NY × 5"
- *Dictionary encoding*: Replace repeated strings with integer codes
- *Delta encoding*: Store differences between sorted numeric values

**Vectorised execution:** CPU can process an entire column batch using SIMD instructions rather than evaluating row by row.

**Cache efficiency:** Column values are contiguous in memory — sequential access patterns maximise CPU cache hits.

---

## Schema Design: Star and Snowflake

### Star Schema

The standard for analytical databases. One central **Fact Table** surrounded by **Dimension Tables**.

```
           dim_customer
                │
dim_product ── fact_sales ── dim_time
                │
           dim_store
```

**Fact Table** holds the *events*: sales, clicks, page views. High-volume, append-only. Contains foreign keys to dimensions and the measurable *facts* (amount, quantity, duration).

**Dimension Tables** hold the *descriptive context*: who (customer), what (product), when (time), where (store). Relatively small. Intentionally denormalised to eliminate JOINs at query time.

Why "Star"? The entity-relationship diagram looks like a star — fact table in the centre, dimensions radiating outward.

### Snowflake Schema

Dimension tables are further normalised into sub-dimensions:

```
dim_sub_category
       │
  dim_category ── dim_product ── fact_sales ── dim_time ── dim_month
                                     │
                               dim_customer ── dim_region
```

Why "Snowflake"? The ER diagram branches further off each dimension — like a snowflake's intricate structure.

**Trade-off:** Less storage redundancy, but more JOINs at query time. In practice, **most teams prefer Star** — analytical query speed matters more than dimensional storage efficiency.

### Slowly Changing Dimensions (SCD)

Dimensions aren't static — customers move cities, products change categories. How you handle this is an SCD problem:

| Type | Strategy | Example |
|---|---|---|
| SCD Type 1 | Overwrite — no history | Update customer.city in place |
| SCD Type 2 | Add new row with effective dates | Keep old row, insert new row with valid_from/valid_to |
| SCD Type 3 | Add new column — one version of history | customer.prev_city, customer.curr_city |

SCD Type 2 is the most common in production warehouses.

---

## OLAP Cube Operations

An OLAP Cube is a multi-dimensional conceptual model. Imagine a 3D cube where axes are `Time`, `Product`, and `Region`, and each cell holds a pre-aggregated metric (revenue, units sold). Traditional OLAP engines (Hyperion, SSAS) materialised this literally. Modern columnar warehouses compute it on the fly — fast enough to not need pre-aggregation for most use cases.

### Operations

| Operation | What It Does | Example |
|---|---|---|
| **Slice** | Fix one dimension; view a 2D cross-section | All products, all regions — Q3 only |
| **Dice** | Fix multiple dimensions | Q3 + North America |
| **Roll-up** | Aggregate to coarser granularity | Monthly → Quarterly → Yearly |
| **Drill-down** | Break into finer granularity | Yearly → Quarterly → Monthly → Daily |
| **Pivot** | Rotate axes (swap rows and columns) | Product on rows → Product on columns |

**Materialized Views** are how modern warehouses achieve the same pre-aggregation benefit. Pre-compute `SUM(revenue) GROUP BY product_category` and store the result. Refresh on schedule or incrementally.

---

## Data Lake vs Data Warehouse vs Data Lakehouse

### Data Lake

- Stores raw, unstructured or semi-structured data in native format (JSON, Parquet, CSV, images, logs)
- *Schema on read* — structure is applied only at query time
- Very cheap storage (S3, GCS, ADLS)
- Great for ML training data, raw event logs, exploration
- Risk: without governance, becomes a **data swamp** — untrustworthy, undiscoverable data

### Data Warehouse

- Stores cleaned, structured, modelled data
- *Schema on write* — structure must be defined before data is loaded
- More expensive; queries are fast and reliable
- Great for BI, dashboards, executive reporting
- Risk: two-tier architecture (lake + warehouse) = two copies of data, two pipelines to maintain

### Lakehouse (Modern)

Combines the cheap object storage of a lake with ACID transactions and schema enforcement layered on top via **open table formats**.

Key formats: **Delta Lake** (Databricks), **Apache Iceberg**, **Apache Hudi**

```
Scenario A — Classic Two-Tier

Source ──► Data Lake (raw, S3) ──► Data Warehouse (clean, Redshift)
           [ML / exploration]         [BI / dashboards]

Scenario B — Lakehouse

Source ──► Data Lakehouse (S3 + Iceberg / Delta Lake)
           [ML / exploration  AND  BI / dashboards — unified]
```

Lakehouses enable both ML workloads (raw data) and BI workloads (structured SQL) on a single platform, eliminating duplicate storage and redundant pipelines.

---

## HTAP: Hybrid Systems

**Hybrid Transactional/Analytical Processing** — systems that try to serve both workloads simultaneously.

Examples: **TiDB**, **YugabyteDB**, **SingleStore**, **CockroachDB** (analytical extensions)

Approaches:
- Row storage for transactional writes + a separate columnar store replica for reads (TiDB's TiFlash)
- MVCC-based isolation so analytical reads don't block transactional writes

**Use case:** Real-time analytics directly on operational data without pipeline lag. Useful when even 1-minute staleness is unacceptable (fraud detection, real-time dashboards on live data).

**Trade-off:** Neither as optimised as a dedicated OLTP system nor as fast as a dedicated OLAP system at scale. Operational complexity increases significantly.

---

## Partitioning and Materialized Views

### Partitioning in Analytical Databases

Most warehouses partition by time — by day or month. When a query specifies `WHERE date >= '2026-01-01'`, the engine skips all other partitions entirely (**partition pruning**). This can reduce scanned data by 99%.

Clustering (Snowflake) or secondary partitioning (BigQuery) on additional columns (e.g., `region`, `product_category`) further prunes within partitions.

**Rule of thumb:** Always design time as your primary partition key in a warehouse. Every query should have a time filter.

### Materialized Views

A pre-computed query result stored as a physical table. Instead of recomputing `SUM(revenue) GROUP BY product_category` on every dashboard load, you compute once and serve from the materialised result.

| Dimension | Trade-off |
|---|---|
| Query speed | Much faster — result pre-computed |
| Storage | Extra cost for the materialised result |
| Freshness | Stale until refreshed — needs a refresh schedule or incremental update |
| Maintenance | Schema changes to underlying tables cascade |

---

## Go Code: ETL Pipeline and Star Schema

### ETL Pipeline Pattern

```go
package etl

import "context"

// Record is a generic row moving through the pipeline.
type Record map[string]any

// Extractor pulls raw records from a source system.
type Extractor interface {
    Extract(ctx context.Context) (<-chan Record, error)
}

// Transformer applies one step of business logic to a record.
// Each Transformer is a single concern: normalise, enrich, filter, etc.
type Transformer interface {
    Transform(rec Record) (Record, error)
}

// Loader writes transformed records to the destination warehouse.
type Loader interface {
    Load(ctx context.Context, records <-chan Record) error
}

// Pipeline wires Extractor → Transformers → Loader.
type Pipeline struct {
    Extractor    Extractor
    Transformers []Transformer
    Loader       Loader
}

func (p *Pipeline) Run(ctx context.Context) error {
    raw, err := p.Extractor.Extract(ctx)
    if err != nil {
        return err
    }

    out := make(chan Record, 256) // buffer to smooth out backpressure
    go func() {
        defer close(out)
        for rec := range raw {
            current := rec
            failed := false
            for _, t := range p.Transformers {
                current, err = t.Transform(current)
                if err != nil {
                    // Production: send to dead-letter queue; never drop silently.
                    failed = true
                    break
                }
            }
            if !failed {
                out <- current
            }
        }
    }()

    return p.Loader.Load(ctx, out)
}
```

### Star Schema Structs

```go
package schema

import "time"

// FactSale is the central fact table — high-volume, append-only.
// FK fields reference dimension table surrogate keys.
type FactSale struct {
    SaleID     int64
    CustomerID int64   // FK → DimCustomer
    ProductID  int64   // FK → DimProduct
    StoreID    int64   // FK → DimStore
    DateID     int64   // FK → DimTime
    Amount     float64
    Quantity   int
}

// DimCustomer holds descriptive customer context.
// SCD Type 2: add ValidFrom/ValidTo fields to track historical changes.
type DimCustomer struct {
    CustomerID int64
    Name       string
    Region     string
    Segment    string    // e.g., "Enterprise", "SMB"
    ValidFrom  time.Time // SCD Type 2
    ValidTo    time.Time // SCD Type 2 (zero value = currently active)
}

type DimProduct struct {
    ProductID   int64
    Name        string
    Category    string
    SubCategory string
}

// DimTime enables drill-down and roll-up without date arithmetic at query time.
// Pre-populate for all dates in the analysis horizon (e.g., 2020–2030).
type DimTime struct {
    DateID    int64
    Date      time.Time
    Day       int
    Month     int
    Quarter   int
    Year      int
    IsWeekend bool
}
```

**Why a DimTime table?** Pre-populating time attributes (quarter, fiscal period, is_weekend) eliminates expensive `EXTRACT()` and `DATE_TRUNC()` calls at query time and makes drill-down/roll-up trivial: `GROUP BY dim_time.quarter`.

---

## Trade-Off Tables

### Choosing Storage Model

| Scenario | Prefer |
|---|---|
| High-concurrency user-facing writes | Row-oriented (OLTP) |
| Aggregate queries across millions of rows | Column-oriented (OLAP) |
| Need sub-ms read on single row by key | Row-oriented |
| Need to read 3 of 100 columns across 1B rows | Columnar |
| Real-time analytics on live data | HTAP (TiDB, SingleStore) |

### Choosing Schema

| Scenario | Prefer |
|---|---|
| Fast query speed is paramount | Star schema (fewer JOINs) |
| Dimension tables are very large | Snowflake schema (normalise large dims) |
| Rapid iteration, prototyping | Star schema |
| Storage cost is a constraint | Snowflake schema |

### Choosing Warehouse Architecture

| Scenario | Prefer |
|---|---|
| BI / dashboards on clean structured data | Data Warehouse |
| ML training data, raw log storage | Data Lake |
| Both ML and BI from one system | Data Lakehouse (Iceberg / Delta) |
| Real-time analytics, low latency | HTAP or streaming + warehouse |
| Privacy / compliance filtering pre-storage | ETL |
| Maximum flexibility to re-transform | ELT |

---

## How to Remember This

**OLTP vs OLAP — the cash register vs the accountant**
- OLTP = the **cash register**: every transaction recorded in real-time, one at a time
- OLAP = the **accountant**: looks at all the records and tells you the patterns

**ETL vs ELT — where does the T happen?**
- ETL: Transform **before** loading — old way (warehouse was expensive)
- ELT: Transform **after** loading — new way (warehouse is cheap and powerful)
- Memory hook: ELT = "**E**verything **L**ands **T**hen transforms"

**Star vs Snowflake — simple vs intricate**
- **Star**: simple shape, fewer JOINs, faster queries → prefer for production BI
- **Snowflake**: intricate shape, normalised dimensions, more JOINs → prefer when dims are large

**Columnar storage — think newspaper columns**
- A newspaper column is one topic, start to finish — you read only the column relevant to you
- Columnar DB: read only the data column relevant to your query

**Data Lake vs Warehouse vs Lakehouse**
- Lake = raw water (unstructured, cheap, can become a swamp)
- Warehouse = bottled water (clean, organised, more expensive)
- Lakehouse = filtered tap water at the source (raw storage + clean access, unified)

**OLAP Cube operations: SliDiRoP**
- **Sli**ce → fix one dimension
- **Di**ce → fix multiple dimensions
- **Ro**ll-up → coarser granularity
- **P**ivot → rotate axes (drill-down is the reverse of roll-up)

---

## Interview Cheat Sheet

### Signal Phrases to Use

- "Running analytics on the production OLTP database would cause lock contention and degrade SLOs for user-facing traffic — we need to offload to a dedicated OLAP system."
- "I'd design an ELT pipeline: land raw data in the staging layer first, then transform using dbt inside the warehouse. This preserves raw data for reprocessing."
- "Columnar storage means we only read the columns the query touches — for a 100-column table, a 3-column query reads 3% of the data, and compression ratios on same-typed columns are excellent."
- "We'd use a Star schema: one central fact table for events, surrounded by dimension tables. Fewer JOINs means faster analytical queries."
- "Partition by time as the primary key — every analytical query should have a time filter, so partition pruning drops 99% of scanned data."
- "If they can't tolerate pipeline lag, HTAP is worth exploring — but it's a trade-off, neither as optimised as a dedicated OLTP nor as fast as a dedicated OLAP at scale."

### Red Flags to Avoid

- Saying "just run analytics on the main database" without acknowledging the performance impact
- Confusing a **Data Lake** (raw/schema-on-read) with a **Data Warehouse** (clean/schema-on-write)
- Designing OLAP schemas with a normalised 3NF model — normalisation is for OLTP, denormalisation is for OLAP
- Ignoring data freshness — OLAP systems always have some lag; clarify the acceptable SLO
- Conflating ETL and ELT — know which one the scenario calls for and why

### Common Interviewer Probes

| Probe | Strong Answer Direction |
|---|---|
| "Why not run analytics on production?" | Lock contention, I/O competition, different indexing needs, SLO degradation |
| "ETL or ELT — which would you use?" | ELT for flexibility and modern warehouses; ETL when privacy/compliance requires pre-filtering |
| "Why columnar storage for analytics?" | Read only needed columns, better compression, vectorised execution, cache efficiency |
| "Star or Snowflake schema?" | Star for speed (fewer JOINs); Snowflake when dimensions are very large |
| "How do you handle slowly changing dimensions?" | SCD Type 2 — new row per change with valid_from/valid_to |
| "Data Lake vs Warehouse?" | Lake = raw/cheap; Warehouse = clean/fast; Lakehouse unifies both |
| "How do you reduce query cost in BigQuery/Snowflake?" | Partition by time, cluster on filter columns, use materialised views for common aggregations |
| "What's a Data Mart?" | Domain-scoped subset of the warehouse (Finance, Marketing) — faster queries, tighter access control |

---

## References

- Kleppmann, Martin. *Designing Data-Intensive Applications*, Chapter 3 (Storage and Retrieval) — columnar storage and OLAP internals
- Kimball, Ralph. *The Data Warehouse Toolkit* — Star schema, SCD, and dimensional modelling
- Inmon, Bill. *Building the Data Warehouse* — original Data Warehouse definition and architecture
- [Apache Iceberg documentation](https://iceberg.apache.org/docs/latest/) — open table format for Lakehouses
- [dbt documentation](https://docs.getdbt.com/) — ELT transformation layer
- [BigQuery: Introduction to Optimizing Query Performance](https://cloud.google.com/bigquery/docs/best-practices-performance-overview) — partitioning and clustering in practice
- [Snowflake: Understanding Clustering Keys](https://docs.snowflake.com/en/user-guide/tables-clustering-keys)