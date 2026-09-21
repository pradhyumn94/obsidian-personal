
## What it is
AWS's fully-managed, highly-scalable key-value/NoSQL database. "Fully-managed" = AWS handles provisioning, patching, scaling. Proprietary (not open-source) — internals are only known via AWS docs + the Dynamo paper, so this note focuses more on behavior/usage than internals. Supports ACID transactions (`TransactWriteItems`/`TransactGetItems`, up to 100 items across tables) — the old "NoSQL = no transactions" criticism no longer applies. Ask the interviewer if DynamoDB is allowed before assuming vendor lock-in is fine.

## Data Model
- **Table** → top-level structure, defined by a mandatory primary key. Schema-less: items in the same table can have different attributes (flexible, but no attribute-uniformity enforcement — validate at the app layer).
- **Item** → a row; collection of attributes; max 400KB per item (all attributes included).
- **Attribute** → key-value pair; scalar (string/number/bool) or set types; can be nested.
- JSON is just the transport format — actual on-disk storage format is proprietary.

## Primary Key: Partition Key + Sort Key
- **Partition Key** — hashed to determine physical storage location/partition. Choose based on the most common query pattern + even distribution.
- **Sort Key** (optional) — orders items within a partition; combined with partition key = composite key. Enables range queries/sorting within a partition.
- Example: chat app → `chat_id` (partition) + `message_id` (sort). Use a monotonically increasing ID (Snowflake, UUIDv7, ULID) for the sort key, not a raw timestamp — timestamps aren't guaranteed unique at ms granularity.

### Under the hood
- **Hash partitioning** for partition key: request router + partition metadata service map the hash to a storage node (centralized partition map/placement service, unlike the peer-to-peer ring in the original Dynamo paper). Handles automatic partition split/merge as data grows.
- **B-trees** for sort key: within a partition, items are organized in a B-tree by sort key → efficient range queries.
- Composite key lookup: hash partition key → find node, then traverse the B-tree with the sort key.

## Secondary Indexes
| | GSI (Global) | LSI (Local) |
|---|---|---|
| Partition key | Different from base table | Same as base table |
| Sort key | Optional, different | Different |
| Storage | Separate partitions/replicas | Co-located with base table partitions |
| Creation | Add/remove anytime | Only at table creation; can't be removed without dropping the table |
| Max count | 20 per table | 5 per table |
| Consistency | Eventually consistent only | Eventually or strongly consistent |
| Throughput | Own separate RCU/WCU | Shares base table's RCU/WCU |
| Size limit | None | 10GB per partition key |
| Update propagation | Async | Sync with base table write |

- **GSI** — separate internal table with its own hash partitioning; use when querying by an attribute outside the primary key (e.g., all messages by `user_id` across chats).
- **LSI** — shares base table's partition, separate B-tree per LSI sort key within the partition (e.g., messages in a chat sorted by `num_attachments`). Must be planned at table creation.

## Accessing Data
- **Query** — retrieves by primary key / secondary index key conditions; efficient, supports range queries on sort key. Prefer this.
- **Scan** — reads every item in table/index, paginated; avoid on large datasets.
- **PartiQL** — optional SQL-compatible convenience syntax; translates to the same Query/Scan/Put operations under the hood, not a new engine.
- Reads always pull the *entire item* regardless of `ProjectionExpression` (that only trims network payload, not RCU cost or storage read). Normalize large/rarely-needed attributes (e.g., reviews) into a separate table to avoid paying for them on every read.

## Consistency (CAP)
- Per-request choice via `ConsistentRead` flag — not a table-level setting.
- **Eventually consistent (default)** — any of the 3 replicas can serve the read; may be stale; 0.5 RCU/4KB; AP-leaning/BASE.
- **Strongly consistent** (`ConsistentRead=true`) — routed to the leader replica; always current; 1 RCU/4KB (2x cost); only supported on base table + LSIs, **not GSIs**.
- Under the hood: each partition = 3 replicas (1 leader + 2 followers) via **Multi-Paxos**. Leader writes WAL entry, needs quorum (2/3) ack before acknowledging the write; strong reads go to leader, eventual reads go to any replica.

## Architecture & Scalability
- Auto-sharding: partition splits/redistributes automatically on size/throughput limits; hash partitioning spreads load evenly.
- Replicated across 3 AZs per region automatically (not configurable). **Global Tables** = opt-in cross-region replication for low-latency global read/write (mention this alone is usually enough in an interview).

## Security
- Encryption at rest by default; TLS enforced on all API calls (no config needed).
- IAM for fine-grained access control; VPC endpoints for private access without public internet exposure.

## Pricing / Capacity (useful for back-of-envelope math)
- **On-demand** — pay per request; good for unpredictable load.
- **Provisioned** — specify RCU/WCU, billed hourly; cheaper for predictable load.
- 1 RCU = 1 strongly-consistent read/sec of ≤4KB (or 2 eventually-consistent reads/sec). 1 WCU = 1 write/sec of ≤1KB (rounds up).
- Per-partition ceiling: **3,000 RCU / 1,000 WCU** → ~12MB/s reads, ~1MB/s writes per partition. Use this to estimate partition counts (e.g., 10M writes/sec of small items ≈ 10,000 partitions).

## Advanced Features
- **DAX** (DynamoDB Accelerator) — purpose-built in-memory read cache; microsecond latency; requires swapping in the DAX client SDK (API-compatible, not fully transparent). Read-through + write-through. Caches auto-invalidate only for writes made *through DAX* — direct DynamoDB writes can leave stale cache entries until TTL/eviction. Maintains separate item cache (GetItem/BatchGetItem) and query cache (Query/Scan), both always on. Does **not** cache strongly consistent reads (passthrough to DynamoDB).
- **DynamoDB Streams** — built-in [[Change Data Capture (CDC)]] of inserts/updates/deletes. Use cases: sync to Elasticsearch, real-time analytics (via Kinesis Data Streams → Firehose → S3/Redshift/OpenSearch; Firehose can't read Streams directly), Lambda-triggered notifications/cache updates.

## When to Use
✅ Need high scale/availability, flexible schema, single-digit-ms (or DAX microsecond) latency, willing to model around access patterns, OK with AWS lock-in.

❌ Very high-volume workloads where cost dominates; complex ad-hoc queries/joins/aggregations; heavy reliance on many GSIs/LSIs (signals a relational DB may fit better); need to stay vendor-neutral.

## Interview Angle
- State partition key (+ sort key if needed) explicitly when introducing DynamoDB — it's the core design decision.
- Mention Global Tables for cross-region, DAX for caching, Streams for CDC/cross-store sync as needed — don't over-explain unless probed.
- Know the strong-vs-eventual consistency tradeoff and that it's chosen per-request, not per-table.

## Related Topics
- [[Cassandra]] — contrast: peer-to-peer consistent hashing + gossip vs DynamoDB's centralized partition/placement service; tunable consistency per-query in both, but Cassandra has no native transactions.
- [[Consistent hashing]]
- [[Change Data Capture (CDC)]]
- [[CAP Theorem]]
- [[Sharding]]
- [[DB Indexing]]
