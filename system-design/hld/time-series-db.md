---
Status: 🌳 Evergreen
Created: 2026-06-19
Last Updated: 2026-06-19
---

# Time-Series Databases

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [Why It's Called That](#why-its-called-that)
- [What Problems It Brings](#what-problems-it-brings)
- [Data Model](#data-model)
- [Write Path & Storage: Time-Partitioned Chunks](#write-path--storage-time-partitioned-chunks)
- [Compression](#compression)
- [Downsampling / Rollups](#downsampling--rollups)
- [Cardinality: The Defining TSDB Problem](#cardinality-the-defining-tsdb-problem)
- [Push vs Pull Ingestion](#push-vs-pull-ingestion)
- [Trade-Offs](#trade-offs)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [Go Code Examples](#go-code-examples)
- [How To Remember This](#how-to-remember-this)
- [References](#references)

---

## What It Is

A time-series database (TSDB) stores sequences of timestamped data points — metrics, sensor readings, application events — and is optimized for: very high-volume append-only writes, queries over time ranges with aggregation, and automatic expiry of old data. Examples: Prometheus, InfluxDB, TimescaleDB, M3DB, OpenTSDB.

## Why It Exists

Consider monitoring 10,000 hosts, each emitting 50 metrics every 10 seconds — roughly 50,000 writes per second, almost all inserts, almost never updates, almost always queried as "give me the average over the last hour, grouped by host".

A row-store relational database with B-tree indexes struggles with this shape of workload: every insert touches index pages in roughly random order, the storage format is not compressed for this access pattern, and "delete data older than 30 days" becomes millions of row-level deletes plus index maintenance. Time-series databases are built around the actual shape of this workload rather than a general-purpose one.

## Why It's Called That

The name is literal: a *series* of data points ordered by *time*. The defining design decision is that **time is the primary axis** — storage layout, partitioning, and query patterns are all organized around it. Every other characteristic (compression strategy, retention strategy, indexing strategy) follows from optimizing for that axis.

## What Problems It Brings

- **Cardinality explosion** — the number of unique time series can grow combinatorially with the number of distinct tag values (see below). This is the single most common TSDB-specific failure mode.
- **Loss of precision over time** — long-term storage typically relies on downsampling, so very old data is only available at coarser resolution.
- **Limited general-purpose querying** — joins across unrelated series, or relational-style queries, are awkward compared to an RDBMS.

---

## Data Model

```
A single measurement: "cpu_usage"

Tags (indexed, used for filtering/grouping):
  host   = server01
  region = us-east

Fields (the actual values, NOT indexed):
  usage_user   = 23.5
  usage_system = 4.1

Timestamp: 2026-06-15T10:00:00Z

Series Key = measurement + sorted tag set
  "cpu_usage,host=server01,region=us-east"
  -> every unique series key is tracked as a separate time series
```

Tags are indexed via an inverted index (`tag value -> list of series IDs`) — structurally the same idea as the inverted index used by full-text search engines (`term -> list of documents`), applied to tag values instead of text terms.

## Write Path & Storage: Time-Partitioned Chunks

```
Write Path

1. Incoming point (metric, tags, fields, timestamp)
        |
        v
2. Write-Ahead Log (durability, sequential append)
        |
        v
3. In-Memory Buffer ("active chunk", mutable, indexed by series)
        |
        v   (on size/time threshold)
4. Immutable Chunk on Disk (compressed, time-bounded, e.g. a 2-hour block)
```

```
Disk layout, ordered by time range

Chunk 1: [00:00-02:00) - compressed, immutable, on disk
Chunk 2: [02:00-04:00) - compressed, immutable, on disk
Chunk 3: [04:00-06:00) - compressed, immutable, on disk
Chunk 4: [06:00-08:00) - active chunk, in memory, mutable

Retention = 6 hours
  -> when Chunk 4 closes and a new chunk opens, Chunk 1 is deleted
     as a whole file -- no row-by-row DELETE needed
```

Because storage is partitioned by time at the file level, retention enforcement is a file deletion, not a row-by-row scan-and-delete with index maintenance — a major operational simplification compared to a general-purpose row store.

## Compression

Time-series data compresses unusually well because both timestamps and values change predictably.

### Delta-Of-Delta Encoding (Timestamps)

If samples arrive roughly every 10 seconds, consecutive deltas between timestamps are nearly identical — so the delta *of the deltas* is mostly zero, which compresses to almost nothing.

```
Raw timestamps (s):       1000   1010   1020   1030   1041
Delta (vs prev):             -     10     10     10     11
Delta-of-delta:               -      -      0      0      1

Storage: first timestamp in full, first delta in full,
then delta-of-delta values - mostly 0, encoded in ~1 bit each.
```

### XOR-Based Float Compression (Values)

Facebook's Gorilla paper applies a similar idea to values: consecutive metric readings (CPU%, temperature, etc.) are usually close together, so XOR-ing consecutive float values tends to produce results with mostly leading and trailing zero bits, which can be encoded in very few bits. The underlying reason is the same as for timestamps — values change slowly relative to their bit-width, so the difference between consecutive values is mostly zeros.

## Downsampling / Rollups

```
Resolution tiers (example retention policy)

Raw data        (1s resolution)  -> kept 24 hours
Rollup: 1m avg  (1m resolution)  -> kept 30 days
Rollup: 1h avg  (1h resolution)  -> kept 1 year

Query for "last 10 minutes" -> served from raw data
Query for "last 6 months"   -> served from 1h rollup
```

Downsampling reconciles "retain years of history" with "don't store years of raw samples": coarser aggregates are precomputed and retained for longer, while raw data is allowed to expire. Designing the resolution/retention tiers is a common element of "design a monitoring system" prompts.

## Cardinality: The Defining TSDB Problem

**Cardinality** is the number of unique time series — unique combinations of metric name and tag values. Every unique combination requires its own entry in the tag index and its own compressed data stream.

```
metric: http_requests_total
tags: method (GET/POST/...), status_code (200/404/500/...), endpoint (~50 routes)

cardinality so far: ~3 x 5 x 50 = 750 series  -- fine

Now add a tag: user_id (1,000,000 distinct users)

cardinality: 750 x 1,000,000 = 750,000,000 series  -- index explosion
```

Every new series adds an entry to the in-memory tag index, so high cardinality grows that index — and the memory it consumes — without bound, regardless of total data volume. The standard mitigation: keep high-cardinality identifiers (user IDs, request IDs, raw IPs) in logs or traces, which are systems designed for that access pattern, and keep TSDB tags low-cardinality (host, region, status class, endpoint template).

## Push vs Pull Ingestion

| Pull (Prometheus) | Push (InfluxDB / Graphite / StatsD) |
|---|---|
| TSDB scrapes targets on an interval | Applications push metrics to the TSDB |
| Easy to detect "target down" (scrape fails) | TSDB must absorb bursty/uneven ingestion |
| Requires service discovery to find targets | No metrics endpoint needs to be exposed |
| Simpler firewall rules (TSDB -> app, one direction) | Works well for short-lived/batch jobs with no long-lived target to scrape |

---

## Trade-Offs

| Aspect | TSDB | RDBMS |
|---|---|---|
| Write pattern | Append-only, very high throughput | Mixed read/write |
| Storage | Time-partitioned chunks + specialized compression | Row-based, B-tree indexes |
| Query pattern | Time-range scans with aggregation | Point lookups, joins |
| Deletion / retention | Drop whole chunk files | Row-by-row DELETE |
| Sensitivity | Very sensitive to tag cardinality | Not a comparable concern |

---

## Interview Cheat Sheet

**Signal phrases that point here:** metrics, monitoring, observability, IoT sensor data, dashboards, alerting.

**Red flags to avoid:**
- Proposing a general-purpose RDBMS with a timestamp column and B-tree index for high-volume metrics without addressing write throughput, compression, or retention
- Discussing tags without ever mentioning cardinality
- Ignoring retention/downsampling when asked about long-term storage cost

**Common probes:**
- "How do you handle millions of writes per second?" — batched, append-only ingestion into time-partitioned chunks
- "How do you keep storage costs down for years of data?" — downsampling/rollup tiers plus retention policies (drop whole chunks)
- "What happens if you add a `user_id` tag to every metric?" — cardinality explosion; move high-cardinality identifiers to logs/traces instead

---

## Go Code Examples

Delta-of-delta encoding for timestamps:

```go
package main

import "fmt"

// encodeDeltaOfDelta returns the delta-of-delta sequence for monotonically
// increasing timestamps. With a roughly constant sampling interval, most
// resulting values are 0 -- which is why this compresses so well.
func encodeDeltaOfDelta(ts []int64) []int64 {
	if len(ts) < 2 {
		return nil
	}
	deltas := make([]int64, len(ts)-1)
	for i := 1; i < len(ts); i++ {
		deltas[i-1] = ts[i] - ts[i-1]
	}
	dod := make([]int64, len(deltas))
	dod[0] = deltas[0] // first delta stored as-is
	for i := 1; i < len(deltas); i++ {
		dod[i] = deltas[i] - deltas[i-1]
	}
	return dod
}

func main() {
	ts := []int64{1000, 1010, 1020, 1030, 1041}
	fmt.Println(encodeDeltaOfDelta(ts)) // [10 0 0 1]
}
```

---

## How To Remember This

**"Append-only diary, not a filing cabinet."** New entries are written at the end and read by date range; old pages are never edited. This property is why compression works so well (predictable deltas) and why deletion is cheap (ripping out old pages = dropping a chunk file).

**Cardinality = combinatorial explosion of tags.** `metric x tag1_values x tag2_values x ...`. Any tag with effectively unbounded values (user IDs, request IDs, raw IPs) multiplies the series count by that many — the number to sanity-check before adding any tag.

**Delta-of-delta = "did the gap between samples change?"** With regular sampling, the answer is almost always "no" — and "no" compresses to nothing.

---

## References

- Prometheus documentation — storage internals (TSDB)
- "Gorilla: A Fast, Scalable, In-Memory Time Series Database" — Facebook, VLDB 2015
- InfluxDB and TimescaleDB documentation
- *Designing Data-Intensive Applications* — Martin Kleppmann