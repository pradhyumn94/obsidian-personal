# Cassandra — Concise Notes

## What it is
Apache Cassandra is an open-source, distributed, wide-column NoSQL database (Dynamo + Bigtable heritage). Built at Facebook for inbox search; used by Discord, Netflix, Apple, Bloomberg. Optimized for horizontal scale, high write throughput, and availability over strict consistency.

## Data Model
- **Keyspace** → database-level container; defines replication strategy.
- **Table** → rows + columns within a keyspace.
- **Row** → identified by primary key.
- **Column** → name/type/value + write timestamp; sparse (rows can have different columns); conflicts resolved via **last-write-wins**.
- **Primary Key** = Partition Key (which node/partition) + optional Clustering Key(s) (sort order within partition).

Query-driven modeling, not entity-relationship: no JOINs, no foreign keys, denormalization is expected.

## Partitioning
- Uses **consistent hashing**: data hashed onto a ring; walk clockwise to first node.
- Avoids mass re-mapping when nodes join/leave (unlike `hash % n`).
- **Vnodes**: each physical node owns many virtual ring positions → even load distribution, supports heterogeneous hardware.

## Replication
- Replication factor (RF) copies stored on next N distinct physical nodes clockwise from the hash point.
- **SimpleStrategy**: naive ring-based (dev/test).
- **NetworkTopologyStrategy**: rack/datacenter aware (production).

## Consistency
- Tunable per read/write: `ONE`, `QUORUM` (n/2+1), `ALL`, etc.
- `QUORUM` reads + writes guarantee overlap → read sees latest write.
- No ACID/transactions; only atomic/isolated writes at row-in-partition level. Aims for eventual consistency.

## Query Routing
- Any node can be a **coordinator**; nodes learn cluster state via **gossip** and route to the correct replica set.

## Storage Engine (LSM Tree)
Write-optimized; writes are near-append-only:
1. Write → **Commit log** (WAL, durability)
2. Write → **Memtable** (in-memory, sorted by key)
3. Memtable flushed → **SSTable** (immutable, on-disk, sorted)
4. Commit log entries pruned after flush
- Reads: check Memtable → bloom filter to find candidate SSTables → scan newest→oldest.
- **Compaction** merges SSTables, purges tombstones (deleted rows).
- **Deletes** = tombstones, resolved during reads/compaction.

## Gossip & Fault Tolerance
- **Gossip**: peer-to-peer state exchange (liveness, schema) using generation/version vector clocks; biased toward **seed nodes** to avoid partitioned sub-clusters.
- **Phi Accrual Failure Detector**: per-node probabilistic failure detection → "convicts" unresponsive nodes.
- **Hinted handoff**: coordinator temporarily buffers writes for a down replica, replays them once it's back.

## Data Modeling Patterns — Discord & Ticketmaster

### Discord — Message Storage
**Access pattern:** most recent messages in a channel, reverse-chronological, single-partition reads.

v1:
```sql
CREATE TABLE messages (
  channel_id bigint,
  message_id bigint,
  author_id bigint,
  content text,
  PRIMARY KEY (channel_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```
- `channel_id` partition key → all messages for a channel on one partition (no scatter-gather).
- `message_id` = Snowflake ID (chronologically sortable, collision-free) instead of a `created_at` timestamp.

Problem: busy/long-lived channels → unbounded partition growth (Cassandra performs poorly on very large partitions).

v2 — time bucketing:
```sql
CREATE TABLE messages (
  channel_id bigint,
  bucket int,
  message_id bigint,
  author_id bigint,
  content text,
  PRIMARY KEY ((channel_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```
- `bucket` = fixed 10-day window (aligned to Discord's custom `DISCORD_EPOCH`, Jan 1 2015), added into the partition key → caps partition size, and new buckets = new partitions as time passes.
- Most reads still hit a single partition (current bucket); exceptions are bucket-boundary crossings or old/inactive channels.

### Ticketmaster — Ticket Browsing UI
**Access pattern:** browse one event's seats at a time, tolerant of weak consistency (real availability re-checked at checkout), UI drills venue → section → seat.

v1:
```sql
CREATE TABLE tickets (
  event_id bigint,
  seat_id bigint,
  price bigint,
  PRIMARY KEY (event_id, seat_id)
);
```
- Single partition per event → for large venues (10k+ seats) this partition gets large, and on-demand aggregates (availability count, price range) are expensive under heavy read load.

v2 — split by section:
```sql
CREATE TABLE tickets (
  event_id bigint,
  section_id bigint,
  seat_id bigint,
  price bigint,
  PRIMARY KEY ((event_id, section_id), seat_id)
);
```
- `section_id` in the partition key spreads an event's seats across partitions/nodes and matches the UI's section drill-down.

Denormalized summary table for the venue-level view:
```sql
CREATE TABLE event_sections (
  event_id bigint,
  section_id bigint,
  num_tickets bigint,
  price_floor bigint,
  PRIMARY KEY (event_id, section_id)
);
```
- Precomputes per-section counts/price floor so the UI never aggregates the full `tickets` table; `event_id` partition key, low section cardinality (<100) keeps it single-partition.
- Staleness is acceptable — Ticketmaster's own UI only shows rounded totals (e.g., "100+").

### Shared principles
1. Model for queries, not entities — no JOINs, so UI access patterns dictate the primary key.
2. Partition key must bound partition size — add a bucketing/sharding dimension (time, section, etc.) if the natural key alone leads to unbounded/oversized partitions.
3. Denormalize freely — precompute into secondary tables for cheap reads, accepting eventual consistency where tolerable.

## Advanced Features
- **SAI** (Storage Attached Indexes): global secondary indexes, more flexible but slower than partition-key queries.
- **Materialized Views**: Cassandra auto-maintains denormalized derived tables.
- **Search integration**: via Elasticsearch/Solr plugins (e.g., Stratio Lucene Index).

## When to Use
✅ High write throughput, massive horizontal scale, flexible/sparse schemas, availability > consistency, clear/limited access patterns.

❌ Strong consistency needs, complex ad-hoc queries, multi-table JOINs/aggregations.
