### CQRS (Command Query Responsibility Segregation)
- Split the **write model** (commands, validation, business rules) from the **read model** (queries), instead of one model serving both.
- Write side stays normalized and optimized for correctness; read side is denormalized and optimized for the exact query shape the UI needs.
- Read and write models can scale independently, use different databases entirely (e.g. Postgres for writes, Elasticsearch or a denormalized cache for reads), and be deployed/scaled separately.
- The two sides sync via events or a projection pipeline — the read model is eventually consistent with the write model, not updated synchronously in the same transaction.

### Event Sourcing
- Instead of storing current state (a row that gets overwritten), store the full sequence of **immutable events** that led to that state (`OrderPlaced`, `OrderShipped`, `OrderCancelled`...).
- Current state is derived by replaying events from the beginning (or from the last snapshot).
- **Snapshots**: periodically persist derived state so you don't replay the entire event history on every read — replay only events since the last snapshot.
- The event log is the source of truth; any read model (SQL table, cache, search index) is just a projection that can be rebuilt from it.

### Why they're usually paired
- CQRS needs some way to keep the read model in sync with the write model — event sourcing provides exactly that stream of events to project from.
- You can do CQRS without event sourcing (sync via CDC or plain domain events), and event sourcing without CQRS (single model, just event-backed) — but combined, the write side appends events and the read side is a set of projections subscribed to that stream.

### Advantages
- Full audit trail for free — every state change is recorded, not just the latest value.
- Temporal queries: reconstruct "what did this look like at time T" by replaying up to that point.
- Read models can be tailored per use case and rebuilt/reindexed at will since they're just projections.
- Decouples read and write scaling — a read-heavy feature doesn't force write-side schema compromises.

### Trade-offs
- Read model lag — clients can briefly see stale data right after a write (eventual consistency).
- Significant complexity: event schema versioning/migration, replay performance, debugging "what does the read model currently reflect" across projections.
- Not a good fit for simple CRUD with no auditing/temporal need — it's a targeted tool, not a default architecture.

### Interview angle
- Good answer for "how would you support undo / audit history / analytics on write-heavy data" questions.
- Mention snapshots explicitly — interviewers probe whether you know naive full-replay doesn't scale.
- Be ready to name the consistency cost: it's a deliberate CAP-style trade (favor availability/scalability of reads over read-after-write consistency).

See also: [[CAP Theorem]] (read model lag is an explicit consistency trade-off), [[Kafka fundamentals]] (event log/stream is commonly backed by a log like Kafka), [[Scaling Reads]], [[Handling Contention]]
