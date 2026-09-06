In-memory key-value store used for [[Caching|caching]], session storage, real-time analytics, leaderboards, rate limiting, and event sourcing.

### Pros
- High performance: sub-ms latency, 100k+ writes/sec
- Versatile data structures: strings, hashes, sets, sorted sets, streams, geospatial
- Simple commands, easy to reason about
- Scales horizontally via clustering ([[Sharding]]) and replication
- Durability options (RDB, AOF) trade off persistence vs performance

### Cons
- Memory-intensive — stores everything in RAM, costly at scale
- Data loss risk if persistence isn't configured
- Manual eviction policy tuning needed for memory overflow
- Not suited for complex queries or relational data

### Use cases
1. **Caching** — `SET`/`GET` with TTL for automatic eviction (e.g. product detail caching)
2. **Session management** — `HSET` for session data, `EXPIRE` for expiry
3. **Real-time analytics** — `XADD`/`XREAD` streams for continuous event logging
4. **Leaderboards** — `ZADD` to rank, `ZRANGE` to read top-N, `ZREMRANGEBYRANK` to trim
5. **Rate limiting** — `INCR` counter + `EXPIRE` window, reject once over limit
6. **Proximity search** — `GEOADD`/`GEORADIUS` for nearby-driver style lookups
7. **Pub/Sub & event sourcing** — `PUBLISH`/`SUBSCRIBE` for real-time notifications; `XCLAIM` lets workers reclaim unprocessed stream entries on failure — compare to [[Kafka fundamentals|Kafka]]'s consumer groups for the same problem at higher throughput
8. **Fraud detection** — hashes/streams for fast real-time aggregation

### Related
- [[Scaling Reads]] — where Redis fits as the application cache layer
- [[Consistent hashing]] — how Redis Cluster distributes keys and handles hot spots
- [[CAP Theorem]] — Redis defaults to availability over strong consistency across replicas
