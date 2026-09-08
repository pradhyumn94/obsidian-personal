Default choice in system design interviews unless requirements demand otherwise. Start with Postgres, then justify deviating — not the reverse.

## Read performance
See [[DB Indexing]] for index internals in depth.
- **B-tree** (default) — exact matches, range queries, `ORDER BY` when it matches index column order
- **GIN** — full-text search (`tsvector`/`to_tsquery`) and JSONB containment queries (`@>`). Often removes the need for a separate Elasticsearch or Mongo — reach for those only when you need fuzzy matching, faceted search, or distributed search at very large scale
- **GiST** (via PostGIS) — geospatial queries (`ST_DWithin`, bounding-box lookups). Uber's ride-matching started on this before moving to custom infra
- **Covering index** (`INCLUDE`) — stores extra columns in the index so the query never touches the table; costs space + slower writes
- **Partial index** (`WHERE status = 'active'`) — index only the subset of rows actually queried; smaller and faster than indexing everything
- Don't over-index: every index slows writes, costs disk, and may be skipped by the planner anyway

### Rough numbers
- Simple indexed lookup: 50k+/sec/core
- Joins with indexes: thousands–tens of thousands/sec
- Full-table scans: dominated by whether the working set fits in RAM
- Tables get unwieldy >100M rows; joins get hard >10M rows — see [[Infra numbers to know]]

## Write performance
1. Write → dirty page + WAL record in memory
2. Commit → WAL flushed to disk (sequential, fast) — this is what actually gates commit latency
3. Background writer flushes dirty pages to data files later, batched
4. Each index adds its own WAL entries — more indexes = slower writes

**Throughput** (single node, good hardware): ~5k simple inserts/sec/core, ~1-2k updates/sec/core, hundreds/sec for complex multi-table transactions. Bottleneck is WAL disk I/O. Connections are OS processes — use a pooler (PgBouncer) once you have many app instances.

### Scaling writes past a single node
See [[Scaling Writes]] for the general playbook (batching, load shedding, hierarchical aggregation).
1. **Vertical scaling** — faster NVMe for WAL, more RAM for buffer cache, more cores
2. **Batching** — collect writes into one transaction; risk is losing the whole batch on crash
3. **Write offloading** — route non-critical writes (analytics, "last seen") through a queue ([[Kafka fundamentals|Kafka]]) to async workers instead of the primary
4. **Table partitioning** (`PARTITION BY RANGE`) — e.g. by month; confines index updates and scans to the relevant partition, lets you tier storage (hot partitions on NVMe, cold on cheap disk)
5. **Sharding** — Postgres has no built-in sharding (unlike DynamoDB); shard key should match your dominant query pattern to avoid cross-shard scatter-gather. Use Citus for managed sharding. See [[Sharding]]

## Replication
- **Asynchronous** (default) — primary confirms write immediately, replicates in background; fast, but a crash can lose the unreplicated window
- **Synchronous** — primary waits for ≥1 replica ack before confirming; stronger durability, added write latency. Common pattern: a few sync replicas for durability + more async replicas for read scaling
- **Read replicas** scale read throughput ~Nx but introduce replication lag → breaks read-your-writes consistency if a user reads from a lagging replica right after writing
- **High availability**: replica promotion on primary failure — usually handled by managed services (RDS, Cloud SQL); know that it's possible, not the manual mechanics

## Consistency
ACID compliant, but ACID alone doesn't solve concurrency — you still need to choose a mechanism. See [[CAP Theorem]] for how ACID-consistency differs from CAP-consistency.

- **Row-level locking** (`SELECT ... FOR UPDATE`) — locks specific rows; use when you know exactly which rows need atomic read-then-write (e.g. auction bid check-and-update)
- **Serializable isolation** — makes transactions behave as fully sequential; simpler to reason about but requires retry logic on conflict. Prefer when the transaction touches too much to reason about explicit locks
- **Optimistic concurrency control** — read a version/timestamp column, write conditionally on it being unchanged, retry on 0-row update. No native syntax; implement at the app level. Best when conflicts are rare
- **Isolation levels** (Read Committed default → Repeatable Read → Serializable): Postgres's Repeatable Read is stronger than the SQL standard — it also blocks phantom reads, which other DBs allow at that level

## When to reach for something else
- **Extreme write throughput** (millions/sec) — Cassandra for event streaming, Redis for real-time counters — see [[Redis]]
- **Active-active multi-region writes** — Postgres is single-primary; use CockroachDB, Cassandra, or DynamoDB global tables instead
- **Pure key-value access** — MVCC/WAL/planner overhead isn't worth it; use Redis or DynamoDB
- Scale alone is not a good reason to abandon Postgres — partitioning, sharding, and replication go a long way first
