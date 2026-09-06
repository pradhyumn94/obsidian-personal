Companion to [[Scaling Reads]] — write-heavy systems trade off differently: fewer indexes, different storage engines, and sharding by write pattern rather than read pattern.

## 1. Storage engine and DB choice
- Cassandra-style DBs favor write throughput at the cost of read performance
- **Time-series DBs** (InfluxDB, TimescaleDB) — built for high-volume sequential timestamped writes, with delta encoding for storage efficiency
- **Log-structured DBs** (LevelDB) — append new data rather than updating in place
- **Column stores** (ClickHouse) — batch writes efficiently for analytics workloads
- **Disable expensive features** during high-write periods — foreign key constraints, complex triggers, full-text search indexing
- **Tune write-ahead logging** — batch multiple transactions before flushing to disk (e.g. PostgreSQL)
- **Reduce index overhead** — fewer indexes speed up writes at the cost of read performance (see [[DB Indexing]])

## 2. Sharding and partitioning
See [[Sharding]] for shard key selection, hot-spot handling, and cross-shard tradeoffs in depth.
1. **Horizontal sharding** — split rows
	- Choose a good partitioning key
	- Consistent hashing, virtual nodes, and slot assignment schemes — see [[Consistent hashing]]
2. **Vertical partitioning** — split columns
	- Separate data by access pattern and scaling requirements instead of cramming everything into one table

## 3. Queues and load shedding
- **Short-lived bursts** — use queues when you expect bursts that are short-lived, not to patch a database that can't handle the steady-state load
- **Load shedding** — drop less important writes to keep the system running and let the more important writes still get processed
	- E.g. location data for drivers and users in ride-sharing apps

## 4. Batching and hierarchical aggregation
Rather than accepting writes as given, change the shape of incoming data so it's cheaper to process. Individual writes pay overhead (network round trips, transaction setup, index updates) and most DBs process batches more efficiently than singletons — so look upstream of the DB, not just at it.

### Batching
Amortize per-write overhead by grouping writes together, at one of three layers:
1. **Application layer** — client batches writes before sending to the DB
	- Works well when the app isn't the source of truth (e.g. consuming from [[Kafka fundamentals|Kafka]], processing, then writing) — a crash just means re-reading the topic, no data lost
	- If the app *is* the source of truth, a crash before the batch flushes loses confirmed writes — not all systems can tolerate this
2. **Intermediate process** — a batcher sits between source and DB and tabulates before forwarding
	- E.g. a "Like Batcher" reads raw like events off Kafka, tallies counts per post over a window, and forwards one aggregated delta per post instead of one write per like — 100 likes/min collapses to 1 write/min
	- Batching efficacy depends on actual traffic shape — a 1-minute batch window is useless if most posts only get 1 like/hour
3. **Database layer** — tune how often the DB flushes writes to disk (e.g. Redis defaults to a 100ms flush interval)
	- The "big hammer" — coarse, blunt, reserve for extreme cases

### Hierarchical aggregation
For high-volume streaming data (analytics, live events) where only aggregated views matter, not individual events — process in stages, reducing volume at each step, built up incrementally.

- **Motivating example**: live video stream comments/likes — millions of viewers both producing events (like/comment) and needing to receive every other viewer's events. Naively this is an all-to-all fan-in/fan-out.
- **Fix fan-out**: assign viewers to **broadcast nodes** via [[Consistent hashing|consistent hashing]]. Write once to each broadcast node, which forwards to its assigned viewers — N viewers becomes M broadcast nodes.
- **Fix fan-in**: route incoming events through **write processors** keyed by comment ID (or the comment ID being liked). Each write processor aggregates likes/comments it owns over a window and forwards a batch of deltas to a **root processor**, which just merges them — instead of the root processor absorbing every raw event directly.
- Net effect: number of writes any single node handles drops substantially, at the cost of added latency from the extra hops.

## Common deep dives
1. How do you handle re-sharding when you need to add more shards?
	- The dual-write phase ensures no data is lost during migration
2. What happens when you have a hot key that's too popular for even a single shard?
	- **Split all keys** — e.g. the `post1Likes` key becomes `post1Likes-0`, `post1Likes-1`, ... `post1Likes-k-1`
	- **Split hot keys dynamically** — only splits keys detected as hot, but creates a mismatch: writers spread across sub-keys while readers may not read all of them
		- Readers always check all sub-keys — same read amplification as the static split, but simple to implement. Writers detect a key is hot (via local stats) and conditionally write to sub-keys.
		- Writers announce the split to readers first — more efficient (readers skip sub-keys that don't exist yet), but requires all readers to receive the announcement before the split takes effect
