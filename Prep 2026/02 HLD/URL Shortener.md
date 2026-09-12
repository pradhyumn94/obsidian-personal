![[Pasted image 20260912144320.png]]

## Deep Dive / Senior-Staff Follow-ups

### 1. ID Generation (core of this problem)
- "Global Counter" as drawn is a single point of contention/failure at 1B+ scale.
- Alternatives:
  - Base62 encode a distributed counter — app servers pre-allocate ranges (e.g. blocks of 1000 IDs) from Redis `INCR`/Zookeeper to avoid hitting the counter per request
  - Snowflake-style IDs (timestamp + worker ID + sequence) — no central coordination needed
  - Random 7-char generation + collision check — probabilistic, needs retry logic
- Probe: what happens when the counter service needs to scale or fails over? How do you avoid ID collisions across counter replicas?

### 2. Custom Alias & Uniqueness
- Need atomic uniqueness enforcement — DB unique constraint on short_code/alias
- Race condition: two users request the same alias simultaneously → return 409, not silent overwrite

### 3. Database Choice & Schema
- SQL vs NoSQL: at 1B rows with simple key lookups (short_code → URL), a KV store (DynamoDB, Cassandra) often beats relational
- Partitioning/sharding key — shard by hash(short_code) since a single DB instance won't hold 1B rows at low latency
- Secondary indexes needed if listing "my URLs" per user for analytics/dashboard

### 4. Read/Write Ratio & Caching Depth
- Bitly-style traffic is read-heavy (100:1+)
- Cache eviction policy (LRU), TTL vs expiration semantics, cache stampede protection for hot links
- CDN edge caching for 302 redirects (short TTL) to reduce origin load further

### 5. Expiration Handling
- Schema has `optionalExpiry` but no mechanism shown for cleanup
- Options: TTL indexes (Mongo/DynamoDB TTL), background reaper job, or lazy expiration-check-on-read
- Tradeoff: eager vs lazy cleanup cost at 1B rows

### 6. Analytics (called out in problem statement, missing from HLD)
- Click tracking needs an async event pipeline (Kafka/Kinesis) off the hot redirect path so writes don't block reads
- Aggregation via stream processing or batch job
- Separate analytics store (ClickHouse/Druid) rather than the hot-path DB

### 7. Consistency Model Justification
- NFR says "availability >> consistency" — be ready to defend the edge cases
- What if two DB replicas briefly disagree on whether an alias is taken? Idempotent writes, or strongly consistent write path + eventually consistent reads?

### 8. Security / Abuse
- Rate limiting on `/shorten-url` to prevent spam link generation
- Malicious URL / phishing detection (Safe Browsing API) before shortening
- Open redirect risk — validate destination URL scheme/format

### 9. Multi-Region / Disaster Recovery
- Global counter + single DB as drawn implies single-region
- Extension: active-active multi-region writes, region-prefixed IDs to avoid cross-region collisions, CDN/edge routing for global low-latency redirects

### 10. Capacity Math
- Have scale numbers (1B URLs, 100M DAU) but no worked-through estimate
- Derive: writes/sec, reads/sec, storage growth/year, cache size needed to cover hot-key working set — use these to justify architecture choices rather than asserting them