# Design a Local Delivery Service like Gopuff

![[Excalidraw/Local delivery service.excalidraw.md]]

## Requirements
- **NFRs:** 100k items, 10k fulfillment centers, 1M orders/day; availability reads <100ms
- **Consistency:** availability reads = eventual (ok, small stock lag acceptable while browsing); ordering = strong (prevents overselling a single physical item)
- Out of scope: payments, drivers, cancellations/returns
- Interview tip: always pair a consistency claim with the *why* (e.g. "ordering must be strongly consistent to prevent overselling one item")

## Core Entities
- **Item** — catalog/product type (e.g. Lays)
- **Inventory** — physical instance of an Item at a location (separate from Item since one Item → many Inventory records across centers)
- **Distribution Center** — physical storage location
- **Order** — set of items ordered by a user

## API
```
GET /availability?lat=LAT&long=LONG&items=item1&items=item2&pageSize={}&pageNum={}
POST /order { lat, long, items: [item1, item2, ...] }
  → { status: 'ordered'|'failed', orderId?, trackingLink? }
```
- Define GET response shape explicitly (itemId, availableQuantity, estimatedDeliveryTime) — an incomplete contract is a red flag
- Use repeated query params (`items=a&items=b`) not comma-separated lists — safer with special chars, more idiomatic REST
- Always paginate search/availability endpoints (limit/offset or cursor)

## High-Level Design
- **Availability:** API gateway → availability service → queries nearby DCs (within N miles) → fulfillment service checks item stock at those DCs
- **1-hour delivery filter:** push into the nearby-service — filter candidate DCs by actual travel time via a routing/mapping service, not straight-line distance (roads/traffic can make a farther DC faster). Apply routing once per candidate DC, not per item, to avoid redundant calls
- **Ordering:** gateway → order service → looks up DC via nearby service → single SERIALIZABLE transaction: check inventory → decrement → write order → commit/rollback
  - Keep inventory + order tables in the same DB (e.g. Postgres) to get atomicity for free; separate DBs would need distributed locking
  - Race-condition safety: SERIALIZABLE isolation already reverts the whole transaction if any item fails the inventory check — no extra handling needed
- **Data model:** separate `Orders` and `OrderItems` tables — a single order can contain multiple line items

## Deep Dives
- **Availability scale estimate:** ~1M views/day, 10 views/session, ~5% convert → ~20k availability lookups/sec
- **Partitioning:** group DCs by region ID (e.g. first 4 digits of ZIP) and partition inventory by region — queries hit 1-2 partitions instead of the whole dataset
- **Read replicas:** availability reads tolerate staleness → serve from replicas; orders need strong consistency → must go through the leader
- **Geo cache keys:** raw lat/long has near-zero cache reuse (every coord is unique) — normalize to buckets (2-decimal rounding or geohash/grid cell) so nearby users share cache keys and hit rates go up
- **Cache what:** cache nearest-DC lookup results keyed by normalized region/geohash, not raw coordinates — result changes rarely, so long TTLs work well
- **Shared location service:** extract location resolution into its own service if both availability and ordering paths need it — one place to tune/cache/scale instead of duplicating logic