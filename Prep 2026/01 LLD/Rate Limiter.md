# Rate Limiter - LLD Practice Notes


|Algorithm|Per-Key State|Tradeoff|
|---|---|---|
|**Token Bucket**|(tokens, lastRefillTime)|Allows bursts, smooth refill|
|**Sliding Window Log**|List<timestamp>|Perfect accuracy, high memory|
|**Fixed Window Counter**|(count, windowStart)|Simple, boundary effects|
|**Sliding Window Counter**|(currentCount, prevCount, windowStart)|Balanced accuracy/memory|


### Rate Limiter LLD Practice - August 18, 2026

#### Key Takeaways

**Requirements**

* When designing a rate limiter, the core state model is a map keyed by (clientId, endpoint). Each combination gets its own independent quota tracker. Two different clients hitting the same endpoint never share state, and the same client hitting two different endpoints also never share state. Making this explicit in your rules section shows you have thought through how the data is actually stored and looked up.

* When you define a result object for a system like a rate limiter, include three things: whether the request is allowed, how many requests remain in the current window, and how long the caller should wait before retrying if they were rejected. This gives callers everything they need without requiring a follow-up call.

* For a config-driven system, always define what happens when a config entry is missing at runtime. A safe default, like falling back to a conservative limit for unconfigured endpoints, prevents crashes and shows you have thought about the unhappy path.

**Entities**

* The orchestrator pattern means one class coordinates the system without doing the core work itself. RateLimiter owns the map of algorithm instances and delegates decisions to them, but each Limiter handles its own state. This separation keeps the coordinator clean and the algorithms swappable.

* Value objects like RateLimitResult should be immutable because they represent a snapshot of a decision at a point in time. Making them immutable prevents accidental mutation after the fact and makes them safe to pass around without defensive copying.

* A single-method interface like Limiter with just allow(key) is a strong abstraction boundary. It means the orchestrator only needs to know that something can make an allow decision, not how it does it. This lets you swap in different algorithms like token bucket or sliding window without changing the orchestrator.

* The factory pattern is useful for constructing objects when the exact implementation depends on runtime configuration. In a rate limiter, a factory can read endpoint config and return the right Limiter subclass, plus fall back to a default when no config exists.

**Class Design**

* Each rate limiting algorithm needs its own internal state to function. A TokenBucketLimiter tracks 'currentTokens' and 'lastRefillTime' so it can calculate how many tokens to add since the last request. A SlidingWindowLogLimiter tracks a list of timestamps so it can count how many requests happened in the recent window. Without this state, the 'allow(key)' method has nothing to compute against.

* When designing algorithm variants in an OOP interview, always sketch the concrete class fields alongside the method signatures. The fields reveal the real logic. For example, 'TokenBucketLimiter has fields: maxTokens, refillRatePerSecond, currentTokens, lastRefillTime' tells the interviewer you understand how the algorithm works, not just what it is called.

* Value objects like RateLimitResult should have clearly documented field contracts. For example, 'retryAfterMs is null when the request is allowed and a positive integer when denied' is a contract that prevents bugs in calling code. Mentioning this kind of convention in an interview shows you think about how other developers will use your design.

**Implementation**

* When calling a constructor or function, argument order matters and Python will not warn you if you swap two integers. Always double-check that the order of arguments you pass matches the order the constructor declares them. For example, if a constructor is defined as **init**(self, windowSizeMs, maxRequests), passing (maxRequests, windowMs) silently assigns the wrong values to each parameter.

* Unit mismatches are a common silent bug. If your class stores a rate as 'per millisecond' but you pass in a 'per second' value without converting, your math will be off by a factor of 1000 and nothing will crash to warn you. Always convert at the boundary where data enters the system, for example in the factory, and name your variables clearly like refillRatePerMs so the unit is obvious.

* When looking up a key in a Python dictionary, use dict.get(key) instead of dict(key). Calling a dict like a function with dict(key) raises a TypeError at runtime. dict.get(key) safely returns None when the key is missing, which is exactly what you want for a fallback pattern.

**Extensibility**

* Sliding Window Log vs Sliding Window Counter are two different algorithms. A Sliding Window Log stores one timestamp per request, which is accurate but memory-heavy. A true Sliding Window Counter uses fixed time buckets and weights the previous bucket to approximate the count, using far less memory. If an interviewer asks for one specifically, know which is which.

* TTL-based eviction and LRU eviction are both valid memory management strategies for rate limiter state, but they have different correctness tradeoffs. LRU can accidentally evict an active client just because they were least recently used, which resets their quota and lets them bypass limits. TTL-based eviction is safer because it only removes clients that have been genuinely inactive for a set period.

* The Open/Closed Principle means your existing classes should be closed to modification but open to extension. In a rate limiter design, this means adding a new algorithm by creating a new class that implements the Limiter interface and adding one case to the factory, without touching the core RateLimiter orchestrator at all.

​

#### My Answers

**Requirements**

**Q:** (1) Start by asking clarifying questions to understand what you need to build (2) Then list the final set of requirements on the editor to the right.

> *No answer recorded*

**Entities**

**Q:** What are the main entities in the system? A simple list is fine. These will become your classes, enums, etc.

> *No answer recorded*

**Q:** What are the relationships between your entities? Explain how they connect.

> The rate limiter would be the orchestrator of the system. It would use the factory pattern to build its specific algorithmic limiters. It takes incoming requests, which are identified by endpoint and client ID, and assigns them to the appropriate limiter. After a lookup in a map of limiters, it also owns a collection of algorithmic instances configured per endpoint. It handles fallback to a default configuration when a request has no specific route.\
> A limiter can be an interface rather than a class, because each algorithm will have different per-key state management. It would have a single method, allow, which returns a rate limit result. Each concrete algorithm can implement this interface and use its own techniques to manage state.\
> Then there is the rate limit result. It’s a type interface with three fields: whether the request is allowed, how many requests are remaining in the quota, and how many milliseconds until the request is denied.

**Class Design**

**Q:** Define the interface of your classes. For each class, specify its state (properties/fields) and behavior (methods).

> *No answer recorded*

**Implementation**

**Q:** Implement the `create(configData)` method in `LimiterFactory`. It should inspect the algorithm discriminator field in the config, extract the algorithm-specific parameters, and return the appropriate `Limiter` instance — for example a `TokenBucketLimiter` or `SlidingWindowLogLimiter`. Walk me through how you'd structure that logic.

> Since the config data can be algorithm-specific, we’ll extract the algorithm key from the config data. Then each algorithm’s config will be represented as a separate object. We’ll run a switch statement on which algorithm the config asks us to implement. Based on that, we’ll create new limiters, using either a fixed set of limiters or a sliding window limiter.

**Q:** Implement `allow(clientId, endpoint)` in `RateLimiter`. It needs to look up the right `Limiter` for the endpoint (falling back to `defaultLimiter`), build the per-client key, and delegate to the limiter. What does that look like?

> We'll follow the implementation. Get the client ID and endpoint. Using the endpoint, we'll look up in our existing rate limiters. If we find an implemented rate limiter, we'll allow the request using that. Otherwise, we'll use the default.

**Q:** Implement `allow(key)` for `TokenBucketLimiter`. This is the core algorithm — on each call you need to refill tokens based on elapsed time, attempt to consume one token, and return a `RateLimitResult` with the correct `remaining` and `retryAfterMs`. What state does each key entry need, and how do you compute retry time on denial?

> We'll check if the bucket exists. If not, create the bucket. Check how many tokens we need to add for that. Calculate the difference between the last refill time and now, then divide by 1000 because we need it per second. Then update the working tokens. The limit for working tokens will be the capacity, so take the minimum of capacity and existing tokens plus new tokens that have been added. Update the last refill time to now.\
> Now to handle the request: check if the bucket tokens are greater than one. If they are, reduce it by one. Return with the remaining bucket tokens, allow: true.\
> If there is not at least one token to handle the request, we try again after milliseconds. We'll be null in this case. If the bucket has not been refilled to at least one token yet, we cannot reply to this request or allow this request. So the tokens needed will be one minus the rocket tokens. The try-after will be calculated based on how long it takes for these tokens to be added. So that will be 1 minus the available tokens, times 1,000, divided by the refill rate per second. And we'll return a later amount result with allowed: false and remaining: 0. And the try-after. What we completed above.

**Extensibility**

**Q:** Imagine you need to add a new `SlidingWindowCounter` algorithm to the system. Walk me through exactly which classes you'd need to touch and what changes you'd make to `LimiterFactory`. Are there any parts of your current design that would make this harder than it should be?

> We’ll need to change it in two places. The Rate Limiter factory will add a new switch case for the sliding window counter, and it will return the sliding window counter limiter. Now, the starting window counter will take in a window size and a max request count. For allow, we’ll check whether the last window we are tracking has how many keys there are. If the last window size has reached its capacity of keys or the max request count, we disallow it; otherwise, we allow it.

**Q:** Right now, `TokenBucketLimiter` stores per-client state in its `buckets` dict indefinitely. If your API gateway serves millions of unique `clientId`s over time, what happens to memory? How would you change the design of `TokenBucketLimiter` (and possibly `SlidingWindowLogLimiter`) to handle this, and would those changes affect the `Limiter` interface or `RateLimiter` at all?

> We will need an action policy. The simplest approach is to add timestamp tracking—the last access time—for each key. Periodically, a background thread scans the map and removes entries that haven't been accessed in, say, the last sixty minutes. Alternatively, we can use an LRU cache with a fixed capacity. When we reach the limit, we evict the least recently used client's data. And if we're using a distributed setup, we can use Redis TTL to implement the same thing. It's also built into…

**Q:** Suppose two requests for the same `clientId` on the same endpoint arrive at exactly the same time and your `RateLimiter.allow()` is called concurrently on two threads. Trace through what happens inside `TokenBucketLimiter.allow()` — specifically around the `_getOrCreateBucket()` call and the token check-and-decrement — and describe what could go wrong. What's the minimal change you'd make to fix it?

> *No answer recorded*

​

#### Final Code

```python
# ═══════════════════════════════════════════════════════════════════════════
# REQUIREMENTS
#
# Example (Tic Tac Toe):
#   1. Two players alternate placing X and O on a 3x3 grid.
#   2. A player wins by completing a row, column, or diagonal.
#   Out of Scope: UI, AI opponent, networking
# ═══════════════════════════════════════════════════════════════════════════

REQUIREMENTS:
1. single-process, in-memory solution only.
2. configuration is loaded once at startup and remains static for the lifetime of the system.
3. Configuration is provided as a structured input at startup 
4. Each endpoint config includes the endpoint identifier (e.g., a path like /api/search), 
    - the algorithm to use (e.g., TokenBucket, SlidingWindowLog), and 
    - the algorithm-specific parameters for that algorithm (e.g., capacity and refill rate for TokenBucket, or window size and max requests for SlidingWindowLog).
5. System enforces rate limits by checking clientId against the endpoint's configuration.
    - Each client is tracked independently per endpoint: two different clients hitting the same 
      endpoint maintain completely separate quota states (i.e., state is keyed by (clientId, endpoint)).
6. Return structured result: (allowed: boolean, remaining: int, retryAfterMs: long | null)
7. If endpoint has no configuration, use a default limit

Out of scope
- Distributed state (Redis, etc)
- Dynamic configuration updates
- Metrics, logging and monitoring
- Config validation beyond basic checks
- Thread safety (single-threaded assumed; worth discussing if time allows)


# ═══════════════════════════════════════════════════════════════════════════
# ENTITIES & RELATIONSHIPS
#
# Example (Tic Tac Toe):
#   Game, Board, Player
# ═══════════════════════════════════════════════════════════════════════════

RateLimitResult - Typed inteface/ class (allowed,retryAfterMs,remaining)
- has 3 fields 
    - whether the request is allowed, 
    - how many requests remain in the quota
    - how many milliseconds until retry if denied. 
- Once created, it's immutable


RateLimiter - Class
- Orchestrator of system
- uses a factory pattern to build specific alogorithmic limiters
- takes incoming request(endpoint,clientId) and assigns it appropriate limiter after lookup from a map of limiters
- owns the collection of algorithm instances (one per configured endpoint)
- handle fallback to common default configuration when requests hit endpoints which don't have specific rules


Limiter  - Interface
- Implemeted by alogorithmic limiters
- has a single method: allow(key) which returns a RateLimitResult
- Each concrete algorithm implements this with its own logic and per-key state management approach
- Interface as every algorithm can have diffrent per-key state management


# ═══════════════════════════════════════════════════════════════════════════
# CLASS DESIGN
#
# Example (Tic Tac Toe):
#   class Game:
#     - board: Board
#     - currentPlayer: Player
#     + makeMove(row, col) -> bool
# ═══════════════════════════════════════════════════════════════════════════

class RateLimiter:
    - limiters: dict<string, Limiter>
    - defaultLimiter: Limiter

    + RateLimiter(configs, defaultConfig)
    + allow(clientId, endpoint) -> RateLimitResult

class LimiterFactory:
    + create(configData) -> Limiter

interface Limiter:
    + allow(key) -> RateLimitResult

class TokenBucketLimiter implements Limiter:
    # Per-key state: tracks current token count and last refill timestamp
    - buckets: dict<string, {tokens: float, lastRefillTime: long}>
    - maxTokens: int
    - refillRatePerMs: float  # tokens added per millisecond

    + TokenBucketLimiter(maxTokens, refillRatePerMs)
    + allow(key) -> RateLimitResult
    # On each call: refill tokens based on elapsed time, then consume 1 token if available

class SlidingWindowLogLimiter implements Limiter:
    # Per-key state: ordered log of request timestamps within the current window
    - logs: dict<string, list<long>>  # key -> list of timestamps in ms
    - windowSizeMs: long
    - maxRequests: int

    + SlidingWindowLogLimiter(windowSizeMs, maxRequests)
    + allow(key) -> RateLimitResult
    # On each call: drop timestamps outside the window, allow if log size < maxRequests

class RateLimitResult:
    - allowed: boolean
    - remaining: int
    # retryAfterMs is null when allowed, positive ms value when denied
    - retryAfterMs: long | null

    + RateLimitResult(allowed, remaining, retryAfterMs)
    + isAllowed() -> boolean
    + getRemaining() -> int
    + getRetryAfterMs() -> long | null


# ═══════════════════════════════════════════════════════════════════════════
# IMPLEMENTATION
# ═══════════════════════════════════════════════════════════════════════════

class LimiterFactory:
    def create(configData) -> Limiter:
        algorithm = configData["algorithm"]
        algoConfig = configData["algoConfig"]

        match algorithm:
            case "TokenBucket":
                # Convert refillRatePerSecond to refillRatePerMs to match TokenBucketLimiter's expected unit
                refillRatePerMs = algoConfig["refillRatePerSecond"] / 1000
                return TokenBucketLimiter(
                    algoConfig["capacity"],
                    refillRatePerMs
                )

            case "SlidingWindowLog":
                # SlidingWindowLogLimiter expects (windowSizeMs, maxRequests)
                return SlidingWindowLogLimiter(
                    algoConfig["windowMs"],
                    algoConfig["maxRequests"]
                )

            case _:
                raise ValueError("Unknown algorithm: " + algorithm)


class RateLimiter:
   def allow(self,clientId,endpoint)->RateLimitResult:
        selectedLimiter = self.limiters[endpoint]
        if selectedLimiter:
            return selectedLimiter.allow(clientId)
        return self.defaultLimiter.allow(clientId)


class TokenBucketLimiter:
    def allow(self,key)-> RateLimitResult:
        bucket = self._getOrCreateBucket(key)
        
        now = time.currentTimeMillis()
        elapsed = now - bucket.lastRefillTime
        tokensToAdd = (elapsed * refillRatePerMs)
        bucket.tokens = min(capacity, bucket.tokens + tokensToAdd)
        bucket.lastRefillTime = now
        
        if bucket.tokens >= 1
            bucket.tokens -= 1
            return new RateLimitResult(
                allowed: true,
                remaining: floor(bucket.tokens),
                retryAfterMs: null
            )
        else
            tokensNeeded = 1 - bucket.tokens
            retryAfterMs = ceil((tokensNeeded) / refillRatePerMs)
            return new RateLimitResult(
                allowed: false,
                remaining: 0,
                retryAfterMs: retryAfterMs
            )
    
    def _getOrCreateBucket(self,key)
        if !buckets.contains(key)
            buckets[key] = new TokenBucket(capacity, currentTimeMillis())
        return buckets[key]


# ═══════════════════════════════════════════════════════════════════════════
# EXTENSIBILITY
# ═══════════════════════════════════════════════════════════════════════════

@dataclass
class RateLimitResult:
    allowed: bool
    remaining: int
    retryAfterMs: int

class SlidingWindowCounter:
    """
    True sliding window counter using fixed time buckets.
    Approximates the sliding window by weighting the previous bucket's count
    based on how much of it overlaps with the current window.
    Uses less memory than SlidingWindowLog since we store bucket counts
    rather than individual request timestamps.
    """
    def __init__(self, window_size_ms: int, max_requests: int):
        self.window_size_ms = window_size_ms
        self.max_requests = max_requests
        # key -> {bucket_start_ms: count}
        # We keep at most 2 buckets per key (current + previous)
        self.buckets = {}

    def _get_bucket_start(self, now: int) -> int:
        return (now // self.window_size_ms) * self.window_size_ms

    def allow(self, key: str) -> RateLimitResult:
        now = int(time.time() * 1000)
        current_bucket_start = self._get_bucket_start(now)
        prev_bucket_start = current_bucket_start - self.window_size_ms

        if key not in self.buckets:
            self.buckets[key] = {}

        key_buckets = self.buckets[key]

        current_count = key_buckets.get(current_bucket_start, 0)
        prev_count = key_buckets.get(prev_bucket_start, 0)

        # How far into the current bucket we are (0.0 to 1.0)
        elapsed_in_current = now - current_bucket_start
        prev_weight = 1.0 - (elapsed_in_current / self.window_size_ms)

        # Weighted count approximates requests in the sliding window
        weighted_count = current_count + (prev_count * prev_weight)

        if weighted_count < self.max_requests:
            key_buckets[current_bucket_start] = current_count + 1

            # Evict buckets older than previous to bound memory per key
            for bucket_start in list(key_buckets.keys()):
                if bucket_start < prev_bucket_start:
                    del key_buckets[bucket_start]

            remaining = int(self.max_requests - weighted_count - 1)
            return RateLimitResult(allowed=True, remaining=remaining, retryAfterMs=0)

        # Rate limit exceeded — estimate when the weighted count will drop below limit
        # As time passes, prev_weight decreases, so the window clears at the next bucket boundary
        retry_after = self.window_size_ms - elapsed_in_current
        return RateLimitResult(allowed=False, remaining=0, retryAfterMs=retry_after)

class LimiterFactory:
    def create(configData) -> Limiter:
        algorithm = configData["algorithm"]
        algoConfig = configData["algoConfig"]

        match algorithm:
            case "TokenBucket":
                #existing

            case "SlidingWindowLog":
                # existing
            
            case "SlidingWindowCounter":
                return SlidingWindowCounter(
                    algoConfig["windowMs"],
                    algoConfig["maxRequests"]
                )

            case _:
                raise ValueError("Unknown algorithm: " + algorithm)

```

​