

|Algorithm|Per-Key State|Tradeoff|
|---|---|---|
|**Token Bucket**|(tokens, lastRefillTime)|Allows bursts, smooth refill|
|**Sliding Window Log**|List<timestamp>|Perfect accuracy, high memory|
|**Fixed Window Counter**|(count, windowStart)|Simple, boundary effects|
|**Sliding Window Counter**|(currentCount, prevCount, windowStart)|Balanced accuracy/memory|
