### Design a News Aggregator like Google News

<!-- EXCALIDRAW_PRACTICE practiceId="cmu85pmbk05hk08adq6c2wewv" -->

![[Excalidraw/News aggregator.excalidraw]]

#### Requirements

**Functional**
* Aggregate + display news articles from thousands of publishers
* Users can view a regional feed, paginated/infinite scroll
* Out of scope: auth, personalization/recommendations, notifications, social features (comments/sharing)

**Non-functional**
* 100M DAU, spikes up to 500M (5x) — pair steady-state with a spike multiplier to justify auto-scaling/caching/queue buffers
* Feed load < 200ms
* Availability > consistency (AP over CP, see [[CAP Theorem]]) — feeds can be briefly stale, service must stay up

#### Core Entities
* **Article** — content itself (text, URL, media)
* **Publisher** — produces articles (RSS feed URL, last-scraped timestamp)
* **User** — consumer of the feed

#### API
* `GET /feed?cursor={cursor}&limit={limit}&region={region} -> Article[]`
* Cursor-based > offset-based pagination — avoids skipped/duplicated rows when new items are inserted mid-scroll
* `region` as a query param scopes results without per-region endpoints

#### High Level Design
* **Ingestion**: collection service polls publisher RSS feeds → parses raw XML into normalized fields (title, author, published date, region) → writes to DB
* **Storage**: `articles` + `publishers` tables; media (thumbnails) stored in [[Blob Storage|S3]], DB holds only the S3 key/URL reference
* **Regional scoping**: `region` lives on the article/publisher record so reads are a simple `WHERE region = ...` filter, not routing logic; ingestion + DB can be [[Sharding|sharded]] per region
* **Read path**: client → gateway (auth, routing) → feed read service → DB (or cache) → response, media served via CDN

#### Deep Dives
* **Cursor pagination**: cursor = `(timestamp, article_id)` pair, not timestamp alone — avoids duplicates/gaps when multiple articles share a timestamp. Query: `WHERE timestamp < cursor.ts OR (timestamp = cursor.ts AND id < cursor.id)`
* **Feed speed (<200ms)**: precompute per-region feed in [[Redis]] Sorted Sets, scored by publish timestamp → O(log n) range reads map directly to cursor pagination; DB writes emit CDC events consumed by feed workers to keep cache updated; cap at N most recent articles/region to bound memory
* **Sub-30-min freshness w/o RSS**: webhook option — publishers call our ingestion endpoint on publish; trades off needing publisher buy-in against not depending on poll intervals
* **Thumbnail delivery**: precompute multiple resized variants on ingest (don't rely on publisher availability), store in S3, serve via CDN so clients hit the CDN directly, not the app service
* **Cache scaling (10M concurrent reads)**: read-heavy → add [[Scaling Reads|read replicas]] per regional Redis master (writes go to master, reads round-robin/least-connection across replicas); Sentinel handles failover; ~200-300ms replication lag is acceptable for a news feed
* **Hot region / stampede**: when replicas can't catch up fast enough, use [[Caching|request coalescing]] — one in-flight request per cache key fetches, others wait and share the result, bounding load on backing store during spikes
