---
Status: 🌳 Evergreen
Created: 2026-06-19
Last Updated: 2026-06-19
---

# Search Systems — Elasticsearch & Inverted Index

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [Why It's Called That](#why-its-called-that)
- [What Problems It Brings](#what-problems-it-brings)
- [The Inverted Index](#the-inverted-index)
- [Indexing Pipeline: The Analyzer](#indexing-pipeline-the-analyzer)
- [Near-Real-Time: Segments, Refresh, Translog](#near-real-time-segments-refresh-translog)
- [Cluster Architecture: Shards & Replicas](#cluster-architecture-shards--replicas)
- [Query Flow: Scatter-Gather](#query-flow-scatter-gather)
- [Relevance Scoring: BM25](#relevance-scoring-bm25)
- [Trade-Offs](#trade-offs)
- [Keeping Elasticsearch In Sync With The Primary Database](#keeping-elasticsearch-in-sync-with-the-primary-database)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [Go Code Examples](#go-code-examples)
- [How To Remember This](#how-to-remember-this)
- [References](#references)

---

## What It Is

Elasticsearch (ES) is a distributed JSON-document store and search/analytics engine built on top of **Apache Lucene**. Given a query like "find documents containing 'quick brown' close together, ranked by relevance, faceted by category", it returns results in milliseconds across billions of documents.

It is almost always deployed as a **secondary system** sitting beside a primary datastore — not as the system of record.

## Why It Exists

A relational database query like `WHERE description LIKE '%quick%'` cannot use a B-tree index — it degenerates into a full table scan with substring matching: O(n) per query, no relevance ranking, no stemming, no fuzzy/typo-tolerant matching, no "did you mean".

As full-text search requirements grow (product catalogs, log search, support tickets, autocomplete), the data needs to be reshaped into a structure built for the query "given a word, find every document containing it, ranked by relevance" — the **inverted index**. Elasticsearch is the distributed system (sharding, replication, query coordination, cluster management) wrapped around that structure, built on Lucene which implements the index itself.

## Why It's Called That

- **Inverted Index** — a document naturally maps `document -> words` (like a book's table of contents: chapter -> topics). The inverted index flips this to `word -> documents` (like a book's back-of-book index: word -> page numbers). It is "inverted" relative to how the data is naturally produced.
- **Elasticsearch** — "Elastic" because the cluster scales elastically: nodes can be added or removed and shards rebalance automatically. "Search" because full-text search is its core function, implemented via the embedded Lucene library.

## What Problems It Brings

- **Not ACID** — eventual consistency / near-real-time (default ~1s visibility delay). Never use as a system of record.
- **Operational overhead** — JVM heap sizing, shard/cluster topology management, monitoring cluster health (yellow/red states).
- **Expensive schema changes** — a field's mapping (e.g. `text` vs `keyword`) is largely immutable; changing it usually requires reindexing into a new index.
- **Storage overhead** — the inverted index plus stored fields and doc values typically inflate storage to roughly 1.1-1.5x the raw document size.

---

## The Inverted Index

```
Scenario A: Forward Index (document -> terms)
  Doc1 -> [the, quick, brown, fox]
  Doc2 -> [the, lazy, dog]
  Doc3 -> [quick, brown, dog]

Scenario B: Inverted Index (term -> documents) -- this is what ES stores
  the   -> [Doc1, Doc2]
  quick -> [Doc1, Doc3]
  brown -> [Doc1, Doc3]
  fox   -> [Doc1]
  lazy  -> [Doc2]
  dog   -> [Doc2, Doc3]
```

In practice each postings list (the right-hand side) stores more than document IDs — also term frequency (used for scoring) and term positions (used for phrase queries such as `"quick brown"`).

## Indexing Pipeline: The Analyzer

```
Document (JSON)
     |
     v
Analyzer
  - Tokenizer: split text into terms
  - Lowercase filter
  - Stop-word filter (remove "the", "is", "at"...)
  - Stemmer (e.g. "running" -> "run")
     |
     v
Token Stream (normalized terms)
     |
     v
Inverted Index Update (term -> postings list)
```

This pipeline applies per field, and only to fields mapped as `text`. A `keyword` field (e.g. `status: "ACTIVE"`) skips the analyzer entirely and is stored as a single exact-match term. This `text` vs `keyword` distinction is the most common source of "why doesn't my filter/aggregation work" issues — `text` fields are for full-text search, `keyword` fields are for exact match, filtering, sorting, and aggregations.

## Near-Real-Time: Segments, Refresh, Translog

A Lucene index (one shard) is composed of **immutable segments**. New documents are never written into an existing segment — they accumulate and eventually form a new one.

```
Write Path (single shard)

1. Document write --> in-memory buffer (not yet searchable)
2. In-memory buffer --> also appended to Translog (on-disk, for durability/crash recovery)
3. Refresh (default: every 1s)
   in-memory buffer --> new immutable Segment (now searchable)
4. Flush (periodic, e.g. translog reaches a size limit)
   all segments --> fsynced to disk, Translog cleared
5. Background Merge
   several small segments --> one larger segment (keeps segment count manageable)
```

"Near-real-time" refers to the refresh delay (default 1s) between a write and it being searchable. Updates and deletes are implemented as: mark the old document version as deleted (in a `.del` file) and write the new version as a fresh document — actual reclamation of space only happens during segment merges. This is the same immutable-segment-plus-background-compaction shape as an LSM tree, applied to a search index instead of a key-value store.

## Cluster Architecture: Shards & Replicas

```
ES Cluster (3 nodes, index "products": 3 primary shards, 1 replica each)

Node A (master-eligible, data)
  Holds: P0 (primary shard 0), R1 (replica of shard 1)

Node B (master-eligible, data)
  Holds: P1 (primary shard 1), R2 (replica of shard 2)

Node C (master-eligible, data)
  Holds: P2 (primary shard 2), R0 (replica of shard 0)

Rule: a replica of shard N never lives on the same node as
primary shard N (no single point of failure for any shard).
```

Document routing: `shard = hash(routing_value) % number_of_primary_shards`, where the routing value defaults to the document `_id`. This formula is **why primary shard count is fixed at index creation time** — changing it later would change where every existing document is expected to live. Replica count, by contrast, can be changed at any time (replicas are just additional copies). Plan primary shard count for the eventual data volume, not the current volume; resizing later means reindexing into a new index (often behind an alias for zero downtime).

## Query Flow: Scatter-Gather

```
1. Client sends search query
        |
        v
2. Coordinating Node receives query
        |
        +-- forwards to --> Shard P0 -- returns its top-K matches + scores
        |
        +-- forwards to --> Shard P1 -- returns its top-K matches + scores
        |
        +-- forwards to --> Shard P2 -- returns its top-K matches + scores
        |
        v
3. Coordinating Node merges + re-sorts all top-K lists ("reduce phase")
        |
        v
4. Final top-N results returned to client
```

"Coordinating node" is a role any node can take for a given request — not a dedicated, fixed node.

## Relevance Scoring: BM25

Elasticsearch's default scoring algorithm is **BM25** (successor to TF-IDF). The intuition, without the formula:

- **Term Frequency (TF)** — more occurrences of a query term in a document increase its score, but with diminishing returns (saturating, not linear).
- **Inverse Document Frequency (IDF)** — terms that are rare across the whole index contribute more to the score than common terms ("the" contributes almost nothing; a rare technical term contributes a lot).
- **Field-Length Normalization** — a match within a short field counts for more than the same match within a very long field.

Scoring is tunable: fields can be boosted, and custom scoring functions can be applied on top of BM25.

---

## Trade-Offs

**Elasticsearch vs Relational Database**

| Aspect | Elasticsearch | RDBMS |
|---|---|---|
| Query strength | Full-text, fuzzy, relevance-ranked, faceted | Exact match, joins, transactions |
| Consistency | Eventual / near-real-time (~1s) | Strong (ACID) |
| Schema | Flexible / dynamic mapping | Fixed schema with migrations |
| Role in architecture | Secondary index / search layer | System of record |

**Shard Count Trade-Offs**

| More Primary Shards | Fewer Primary Shards |
|---|---|
| More write parallelism | Less per-shard overhead |
| Spreads large indices across more nodes | Risk of oversized or hot shards |
| Fixed at index creation | Resized only via reindex into a new index |
| More nodes participate in scatter-gather per query | Faster queries on small indices |

---

## Keeping Elasticsearch In Sync With The Primary Database

Because Elasticsearch is almost always a secondary index, the real system design question is: how does data get from the primary database into Elasticsearch?

1. **Dual Writes** — the application writes to the primary database and to Elasticsearch within the same request. Simple to implement, but the two writes are not atomic: if one succeeds and the other fails, the systems drift out of sync.
2. **CDC (Change Data Capture)** — a tool (e.g. Debezium) tails the database's write-ahead log / binlog and streams row changes into Elasticsearch asynchronously. Decouples Elasticsearch availability from the write path entirely; introduces a small replication lag (typically seconds).
3. **Outbox Pattern** — within the same database transaction as the data change, write a row to an "outbox" table representing the event to publish. A separate process reads the outbox and updates Elasticsearch. Avoids dual-write drift without requiring CDC infrastructure.

Whenever a design proposes "index X in Elasticsearch for search", the expected follow-up is "how do you keep it in sync when X is updated?" — pick one of the three above and state its trade-off (CDC: best decoupling, more infrastructure; dual write: simplest, can drift; outbox: middle ground).

---

## Interview Cheat Sheet

**Signal phrases that point here:** full-text search, search-as-you-type / autocomplete, fuzzy or typo-tolerant search, log search and analytics, faceted search and filtering.

**Red flags to avoid:**
- Treating Elasticsearch as the primary source of truth or doing transactional writes against it
- Proposing "index it in ES" without a data sync strategy
- Assuming ACID guarantees or true real-time visibility
- Confusing `text` and `keyword` field semantics

**Common probes:**
- "How do you keep Elasticsearch in sync with your primary database?" — CDC, dual write, or outbox (see above)
- "How would you reindex billions of documents with zero downtime?" — build a new index, backfill, then atomically switch an alias
- "What happens if a node holding a primary shard goes down?" — a replica is promoted to primary
- "How do you scale write throughput?" — more primary shards, but remember this is fixed at index creation

---

## Go Code Examples

A minimal inverted index — the core idea stripped to essentials:

```go
package main

import (
	"fmt"
	"strings"
)

// InvertedIndex maps a term to the set of document IDs containing it.
type InvertedIndex map[string]map[int]bool

// tokenize is a minimal "analyzer": lowercase + split on whitespace.
func tokenize(text string) []string {
	return strings.Fields(strings.ToLower(text))
}

func (idx InvertedIndex) Add(docID int, text string) {
	for _, term := range tokenize(text) {
		if idx[term] == nil {
			idx[term] = make(map[int]bool)
		}
		idx[term][docID] = true
	}
}

func (idx InvertedIndex) Search(term string) []int {
	var docs []int
	for docID := range idx[strings.ToLower(term)] {
		docs = append(docs, docID)
	}
	return docs
}

func main() {
	idx := InvertedIndex{}
	idx.Add(1, "the quick brown fox")
	idx.Add(2, "the lazy dog")
	idx.Add(3, "quick brown dog")

	fmt.Println(idx.Search("quick")) // [1 3]
	fmt.Println(idx.Search("dog"))   // [2 3]
}
```

Using the official client (`github.com/elastic/go-elasticsearch/v8`):

```go
package main

import (
	"context"
	"strings"

	"github.com/elastic/go-elasticsearch/v8"
	"github.com/elastic/go-elasticsearch/v8/esapi"
)

func indexProduct(es *elasticsearch.Client, id, jsonBody string) error {
	req := esapi.IndexRequest{
		Index:      "products",
		DocumentID: id,
		Body:       strings.NewReader(jsonBody),
		Refresh:    "true", // makes the doc searchable immediately; avoid on hot write paths
	}
	res, err := req.Do(context.Background(), es)
	if err != nil {
		return err
	}
	defer res.Body.Close()
	return nil
}
```

---

## How To Remember This

**"Back-of-book index, not table of contents."** A table of contents maps chapter to page (forward index — the natural way to write it down). A back-of-book index maps word to page (inverted index — what Elasticsearch stores). "Inverted" means the direction is flipped from how the data was originally produced.

**"Elastic = the cluster stretches, search = what it's for."**

**Shard count is the foundation poured at construction.** Floors (replicas) can be added anytime. The foundation (primary shard count) cannot be changed without rebuilding (reindexing into a new index).

---

## References

- Elasticsearch official documentation (elastic.co)
- "Elasticsearch: The Definitive Guide" — Elastic
- Apache Lucene documentation
- *Designing Data-Intensive Applications* — Martin Kleppmann, Chapter 3 (storage and indexing structures)