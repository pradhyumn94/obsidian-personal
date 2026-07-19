
Strategies
![[Caching strategies]]


Caching Problems 
1. Cache stampede (Thundering herd)
	 happens when a popular cache entry expires and many requests try to rebuild it at the same time
	How to handle it:
	- **Request coalescing (single flight):** Allow only one request to rebuild the cache while others wait for the result. This is the most effective solution.
	- **Cache warming:** Refresh popular keys proactively before they expire. This only helps when using TTL-based expiration. If you invalidate cache on writes instead, warming does not prevent stampedes.

2. Cache consistency 
	How to handle it:
	- **Cache invalidation on writes:** Delete the cache entry after updating the database so it gets repopulated with fresh data.
	- **Short TTLs for stale tolerance:** Let slightly stale data live temporarily if eventual consistency is acceptable.
	- **Accept eventual consistency:** For feeds, metrics, and analytics, a short delay is usually fine.