Even a well-indexed DB struggles past ~50K-100K reads/sec depending on query pattern — see [[Infra numbers to know]] for the broader per-component thresholds.

## 1. Optimize reads within your DB
1. [[DB Indexing|Indexing]]
	- Avoid full table scans
	- Read drops to O(log(n))
	- Under-indexing is generally worse than over-indexing
2. Vertical scaling
	- Increase size of machine
3. Denormalization ([[Data Modeling]])
	- Intentionally store redundant data to speed up reads
	- Uses more storage
	- Makes writes more complex

## 2. Scale your DB horizontally (replicas/sharding)
1. Read replicas
2. [[Sharding]]
	- Functional sharding
	- Each DB has a slice of the same data

## 3. Caching (application/CDN)
See [[Caching]] for strategies, cache stampede, and invalidation in depth.
1. Application cache
2. CDN and edge cache

### Common deep dives
1. What happens when your queries start taking longer as your dataset grows?
	- Add indexes — see [[DB Indexing]]
2. How do you handle millions of concurrent reads for the same cached data?
	- Hot keys
		- Request coalescing
		- Cache key fan-out
3. What happens when multiple requests try to rebuild an expired cache entry simultaneously?
	- Cache stampede / thundering herd — see [[Caching]]
4. How do you handle cache invalidation when data updates need to be immediately visible?
	- Invalidate cache on write
	- Cache versioning
	- Tradeoffs mirror [[CAP Theorem|consistency vs availability]]
