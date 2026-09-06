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

## 3. Queues/ Load shedding


## 4. Batching/Aggregation
