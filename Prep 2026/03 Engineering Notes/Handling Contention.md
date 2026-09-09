Unhandled concurrent access to shared state causes lost updates, double-bookings, inconsistent state. Distinct from [[Concurrency]] (single-process primitives) — this is contention across requests/services on shared DB state.

## The race
Read-modify-write isn't atomic: two requests read the same stale value, both pass a check, both write — a **lost update** (e.g. two buyers both see "1 seat left," both buy it). Fix: **compare-and-set** — make the write conditional on the value being unchanged since read.

## Conditional writes
Rule is a predicate the DB checks as part of the write — single atomic statement, no lock:
```sql
UPDATE concerts SET available_seats = available_seats - 1
WHERE concert_id = 'weeknd_tour' AND available_seats > 0;
```
Loser matches 0 rows (or aborts under stricter isolation).
- 0-rows-matched isn't an error — gate any dependent `INSERT` on it via `RETURNING`/`INSERT...SELECT`, or check affected-row count
- Guard the actual contended resource, not a proxy — a seat *counter* proves a seat is free, not *which* one; give each seat its own row and guard that
- Elsewhere: DynamoDB `ConditionExpression`, Redis `SET NX`, Cassandra lightweight transactions, HTTP `If-Match`

## Pessimistic locking
For when the decision needs app logic between read and write (not a single `WHERE` predicate) — e.g. scanning for 4 contiguous open seats:
```sql
SELECT ... FOR UPDATE;  -- locks the read rows
-- app picks a block, then writes
```
Failure modes: locking too much/long (serializes writers; never hold a lock across slow I/O like a payment call); deadlocks from inconsistent lock order (fix: always lock in a consistent global order, e.g. by ID; DB auto-detects and aborts one side — retry it). Cost: every transaction pays the lock even when nothing collides.

## Optimistic concurrency control (OCC)
Bets conflicts are rare — no locking, detect collision at write time via a version check:
```sql
UPDATE concerts SET available_seats = available_seats - 1, version = version + 1
WHERE version = 42;  -- version read earlier
```
0 rows → retry. Version source: dedicated counter (safest), timestamp, or a monotonic business value (e.g. auction high bid — must only move one way). **ABA problem**: value goes A→B→A between read/write, equality check misses it — a dedicated incrementing counter is immune, a reused business value usually isn't. Elsewhere: HTTP ETags, etcd revision, DynamoDB version attribute. High contention → lock upfront; low contention (most e-commerce) → optimistic.

## Write skew (isolation levels)
Conflicts spanning rows that never collide directly — no row to lock/version/guard. Classic: on-call rule "≥1 engineer on call," Alice and Bob each see the other still on, both step down, now nobody is. Only `SERIALIZABLE` catches this (guarantees a valid serial ordering exists; none does here → aborts one side).

| Level | Sees | Default in |
|---|---|---|
| READ UNCOMMITTED | uncommitted writes | rarely used |
| READ COMMITTED | committed only | Postgres |
| REPEATABLE READ | stable re-reads | MySQL |
| SERIALIZABLE | as if run one at a time | — |

Expensive (tracks all reads/writes, aborts waste work) — reserve for true cross-row invariants, or materialize the invariant onto one lockable row instead. Most NoSQL stores lack true `SERIALIZABLE`, so folding onto one cell is often the only option there.

## Distributed locks
When exclusivity must outlive one DB transaction — a wait, an external call, multiple steps (e.g. holding a seat 10min during checkout). Hold the lock as **data** (a lease), not a transaction-scoped lock.

| Backing store | Tradeoff |
|---|---|
| Redis `SET NX EX` | Fast, no cleanup — but not airtight (TTL can lapse mid-hold under GC pause → brief double-grant); fine for soft reservations only. SPOF |
| DB column (`reserved_until`) | No new infra; expiry in `WHERE` clause means lapsed = free, no cleanup job. Slower; can hotspot |
| ZooKeeper / etcd | Most robust (Raft/ZAB consensus, ephemeral nodes auto-clean) — real operational overhead |

## Technique equivalents
| Technique | SQL | Elsewhere |
|---|---|---|
| Conditional write | `WHERE` predicate | DynamoDB `ConditionExpression`, Redis `SET NX`, HTTP `If-Match` |
| OCC | `version` column | HTTP ETags, etcd revision, DynamoDB version attribute |
| Pessimistic lock | `SELECT...FOR UPDATE` | mutex / distributed lock |
| Serializable isolation | `ISOLATION LEVEL SERIALIZABLE` | mostly relational-only |
| Distributed lock | reservation row + TTL | Redis `SET NX EX`, ZooKeeper/etcd lease |

Keep the contended resource in one authoritative store — everything above assumes it. Breaks when an op spans multiple services/shards (→ distributed transactions) or a record is writable in multiple places at once (→ conflict resolution: LWW, vector clocks, CRDTs).

## Choosing
| Situation | Approach |
|---|---|
| Predicate on the row being written | Conditional write |
| Read→decide in app code→write | Pessimistic locking |
| Same, but collisions rare | OCC |
| Invariant spans non-colliding rows | `SERIALIZABLE` or materialize onto one row |
| Exclusivity spans a wait/external call | Distributed lock |

Default to the simplest tool; escalate only as the read-write gap grows.

## Deep dives

**Deadlocks?** Opposite lock-acquisition order between transactions → mutual wait forever. Fix: sort resources by a deterministic key, always lock in that order. DB deadlock detection catches the rest — treat the abort as retryable.

**ABA problem?** A→B→A round-trip is invisible to an equality check → stale write commits. Fix: dedicated ever-incrementing version column, or fall back to matching every read field in the `WHERE` clause.

**Hot partition (everyone wants the same row)?** Sharding/load balancing/read replicas don't help — nothing to split when it's all one row. First ask if the problem itself can change (split into parallel items, go eventually consistent). Otherwise, **queue-based serialization**: single worker processes all requests for that resource sequentially — trades throughput for zero contention; back the worker with a standby (now a SPOF).

## When to use in interviews
Call out contention proactively wherever multiple actors compete for one resource: auctions (OCC, monotonic bid as version), booking/Ticketmaster (distributed lock for temp holds), payments (pessimistic/OCC, single-DB only — cross-shard becomes distributed transactions), ride dispatch (TTL'd pending status), flash sales (OCC + TTL cart holds), ratings (OCC with version column). Skip locking/coordination entirely for low-contention or single-user cases (admin-only edits, personal lists) — plain retry logic is enough.

## Related
- [[Concurrency]] — same problem at the single-process/in-memory level
- [[PostgreSQL]] — row locking, serializable isolation, OCC in Postgres specifically
- [[CAP Theorem]] — `SERIALIZABLE`/distributed locks trade availability for consistency
- [[Redis]] — `SET NX EX` as a distributed lock's backing store
