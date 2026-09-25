# Elasticsearch

## Summary
Distributed search engine built as a coordination/orchestration layer on top of Apache Lucene. Handles search-and-retrieval problems (sorting, filtering, ranking, faceting) that go beyond what a Postgres full-text index can do at scale.

## Why it matters
Comes up in any interview question involving complex search (product catalogs, geospatial "near me" queries, log search). Usually attached to an authoritative store (Postgres/DynamoDB) via [[Change Data Capture (CDC)]] rather than used as the system of record. Some interviewers also probe the internals (Lucene segments, inverted index) for infra-heavy roles.

## Core Concepts
- **Document**: a JSON object being indexed/searched (like a row).
- **Index**: a collection of documents, analogous to a DB table.
- **Mapping**: the schema — field names, types (`text`, `keyword`, `float`, `date`, `geo_point`, `nested`, ...) and how each is analyzed/indexed.
  - `keyword` = exact-match/sortable (hash-table-like). `text` = tokenized for full-text match (inverted-index-like).
  - Only map fields you actually search on — extra mapped fields cost cluster memory even if unused.
  - Nested vs. separate index for related data (e.g. book reviews) is a normalize/denormalize tradeoff, same as SQL.

## API Basics
- `PUT /index` — create index (shards/replicas config).
- `PUT /index/_mapping` — define schema up front (vs. dynamic mapping).
- `POST /index/_doc` — add a document.
- `PUT /index/_doc/:id?version=N` — full replace with **optimistic concurrency control** via the `_version` field (reject if version doesn't match).
- `POST /index/_update/:id` — partial update without fetching the whole doc first (important since ES is distributed/async — explicit semantics avoid out-of-order surprises).
- `GET /index/_search` — query DSL is JSON, SQL-like (`match`, `bool`/`must`, `range`, `nested` for nested fields).

## Geospatial Search
- `geo_point` (lat/lon pair) for single locations; `geo_shape` (polygons/lines/circles) for zones/boundaries.
- `geo_distance` query = radius search; combine with `bool` filters (e.g. cuisine + rating) for real product queries (Yelp/Uber-style).
- Under the hood: **BKD tree** (block-optimized k-d tree) indexes `geo_point` — see [[Proximity Search]] for the general spatial-index taxonomy (quadtree/k-d/R-tree/geohash/S2/H3). This is the "custom spatial tree" branch of that note.

## Sorting
- Default sort = relevance `_score` (TF-IDF-based).
- Explicit `sort` param supports multiple fields, custom Painless scripts, and nested-field sort (`mode: max/min` + `nested.path`).

## Pagination
| Method | How | Tradeoff |
|---|---|---|
| `from`/`size` | offset + limit | Simple but O(from) cost server-side; breaks down past ~10k results |
| `search_after` | resume from last doc's sort values | Efficient deep pagination; forward-only, no random page access, no client-side global consistency |
| PIT (point-in-time) + `search_after` | pins a consistent view, then paginate | Adds stateful overhead but immune to results shifting under concurrent writes/deletes |

## Cluster Architecture
- **Node types** (a node can hold multiple roles): Master (cluster coordination, only one active — elected among master-eligible seed nodes), Data (stores shards), Coordinating (receives client requests, fans out, merges results), Ingest (pre-processing pipelines), ML.
- **Index → Shards → Lucene index → Segments.** Shards split data across nodes; each shard is 1:1 with a Lucene index.
- **Replicas** = copies of a shard: give HA and multiply read throughput (X TPS/shard × Y replicas). Coordinating node load-balances reads across primary + replicas.
- Search = two phases: **query** (find matching doc IDs via index structures) then **fetch** (pull `_source` for those IDs). Best queries never need the fetch phase.

## Lucene Segments (the write model)
- Segments are **immutable**. Writes batch into new segments; small segments periodically **merge** into larger ones.
- Deletes: doc is flagged in a per-segment tombstone set (still physically present); reclaimed on next merge.
- Updates: implemented as soft-delete + insert new doc — meaning **updates are more expensive than inserts**. This is why ES is a poor fit for rapidly-mutating counters (like counts/likes).
- Immutability payoff: safe caching, simple concurrency (readers never see data change mid-query), fast crash recovery, better compression.

## Inverted Index & Doc Values
- **Inverted index**: token → list of doc IDs (e.g. `"lazy": [12, 53]`). Turns an O(n) scan for matching docs into an O(1) lookup — the core Lucene/ES full-text-search data structure.
- **Doc values**: columnar, contiguous per-field storage across a segment (solves the "only need one column but row-store forces reading the whole doc" problem — same idea as columnar analytics stores like Redshift). Used for sort/aggregation after the inverted index narrows candidates.

## Query Planning (Coordinating Node)
- Query planner uses field/term statistics (doc frequency, doc length) to choose execution order — e.g. for `"bill nye"`, whether to intersect from the rarer term (`nye`) first vs. the common term (`bill`) can differ by orders of magnitude.
- Classic query-optimization pattern: add statistics + indirection so the system adapts to data shape instead of using a fixed plan.

## Using It in an Interview
- Not a database — no strong durability guarantees; keep the source of truth elsewhere (Postgres/DynamoDB) and sync via CDC.
- Read-heavy tool; avoid for write-heavy/rapidly-mutating fields (see update cost above).
- Eventually consistent — results can be stale; confirm your use case tolerates that.
- Denormalize aggressively (not relational) — aim for 1-2 queries per user-facing search.
- Skip it for small (<100k docs) or rarely-changing datasets — plain DB query is often enough.
- Sync-drift between ES and the source store is a common real bug — call it out as an operational risk.

## Lessons Transferable Beyond Elasticsearch
- Immutability at the storage layer buys caching, compression, and concurrency simplicity — same theme as log-structured storage (see [[PostgreSQL]] WAL/MVCC, [[Kafka fundamentals]]).
- Separating query execution (coordinating nodes) from storage (data nodes) lets you scale/optimize each independently.
- Choice of index structure should match the dominant access pattern (inverted index for text match, doc values for sort/aggregate, BKD for geospatial).

## Related Topics
- [[Proximity Search]]
- [[Change Data Capture (CDC)]]
- [[DB Indexing]]
- [[Sharding]]
- [[CAP Theorem]]
