---
status: Done
category: Fundamentals
date: 2026-10-31
---

# Design a Distributed Rate Limiter - Practice Notes

![[Excalidraw/Distributed Rate Limiter.excalidraw.md]]

See also: [[Rate Limiter|Rate Limiter (LLD)]] · [[API Gateway]] · [[Redis]] · [[Handling Contention]] · [[Zookeeper]].

### Design a Distributed Rate Limiter Guided Practice - October 31, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmvgrdk3508jo01adozyn3vf4" -->

#### Key Takeaways

**Requirements**

* For a distributed rate limiter, eventual consistency is an acceptable tradeoff because enforcing strict consistency across nodes would require expensive coordination that kills performance. A few extra requests slipping through is better than adding latency or reducing availability.

* When scoping a system design, explicitly stating what is out of scope (like strong consistency) is just as valuable as stating what is in scope. It shows you understand the tradeoffs and prevents the interviewer from thinking you simply forgot about it.

* Quantifying scale targets early (like 1M RPS or 10M DAU) anchors all future design decisions. It helps you justify choices like in-memory storage over disk-based storage, or a distributed cache over a single database.

**Core Entities**

* A rate limiter has three core entities. Clients represent who is making the request and can be identified by userId, IP address, or API key depending on the context. Rules define the policy limits such as 100 requests per minute per user. Requests are the incoming traffic that gets checked against the rules to decide if they should be allowed or rejected.

* Client identity in a rate limiter is not always a single field. Using IP address works for anonymous traffic but fails when many users share an IP like in an office. Using userId works for authenticated users but requires a login. API keys are common for third-party developers. Knowing which identifier to use depends on who your users are and whether they are authenticated.

**System Interface**

* A rate limit API response should include three key fields: a passes boolean for the allow or deny decision, a remaining count showing how much quota is left in the current window, and a resetTime timestamp so clients know when to retry. Together these give the caller everything needed to handle rate limiting gracefully.

* Rate limit checks should always take both a clientId and a ruleId as inputs. The clientId identifies who is making the request, and the ruleId identifies which specific limit rule applies, since different endpoints or actions may have different rate limit policies for the same client.

**High Level Design**

* Token bucket has two key policy parameters you should name explicitly: bucket capacity (how much burst you allow) and refill rate (the steady-state throughput you enforce). Bucket capacity controls how many tokens can accumulate when traffic is low, which is why token bucket handles bursts better than a fixed window counter, which resets all at once and can allow double the rate at window boundaries.

* When placing a rate limiter at the API gateway, the gateway must be able to read the client identity you want to limit on, such as API key, user ID, or IP. If you want richer per-user policies like premium tier limits, the gateway either needs access to a fast lookup for that user tier data or the request context must carry that information.

* When a request is rate limited and you return HTTP 429, include a Retry-After header on that same rejected response. This tells the client exactly how many seconds to wait before retrying, so they do not have to compute it themselves from a reset timestamp. It is a small addition that makes your API much easier to integrate against.

**Deep Dives**

* When Redis goes down in a rate limiter, your gateway needs an active health check mechanism to detect the failure. Use a threshold of consecutive failed checks (like 5 in a row) before switching to your fallback policy. This avoids flipping into fail-closed mode on a single transient network blip, which would unnecessarily block legitimate users.

* Each gateway instance should independently track Redis health using its own local health check state, not a central coordinator. This way all instances converge on the same fallback policy around the same time without adding a coordination bottleneck to the failure path.

* In a multi-region rate limiter, each client should always be routed to one specific region where their token bucket state lives. If a client shifts regions due to a traffic change, migrate their state to the new regional Redis rather than doing cross-region lookups on every request. Cross-region reads add significant latency and defeat the purpose of regional deployment. Accepting a brief state reset during migration is a reasonable trade-off.

* For distributing rate limit rule updates across many gateway instances, use a push-based coordination service (like ZooKeeper or etcd watches) rather than polling. Gateways subscribe to rule change notifications and cache the latest rules in memory so they never need to call the coordination service on the hot request path. Also plan for what a gateway should do if it briefly misses a notification, such as retrying the watch subscription.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements?

> Check latency <5ms Availability >> consistency. Eventual consistency will be acceptable across nodes. Handle 1M rps and 10M DAU Out of scope Strong consistency guarantees across all nodes

**Core Entities**

**Q:** What are the core entities?

> Clients:\
> Requests:\
> Rules The entities being rate limited could be users (userId), IP address, API keys, or combinations The incoming API requests that need to be evaluated against rate limiting rules. each request will have userId apikey and IP etc to identify it rate limiting policies that define limits

**System Interface**

**Q:** What is the interface of your rate limiting component? Describe the inputs it needs (e.g., client ID, rule info) and outputs it returns (e.g., allow/deny decision, remaining quota).

> isRequestAllowed(clientId, ruleId) -> { passes: boolean, remaining: number, resetTime: timestamp }

**High Level Design**

**Q:** Where should the rate limiter be placed in your overall system? Consider the trade-offs of different placement options.

> It makes more sense to keep our rate limiting in API Gateway. Because, otherwise. Every service. Will have to. Contain. The checking and the updating logic for our rate limits. Which. Will be, which will not be as fault tolerant. Or. As simple as keeping it in API gateway. But the challenge is. In keeping an API gateway are. We cannot implement. Tough business logic or exact business logic. Let's say you cannot implement. A rate limit saying. Premium users gives 10x the. EPA tokens or the 10x the limit versus non-premium users. You will need something to identify that those users are premium. Or authorized.

**Q:** What rate limiting algorithm would you use and why?

> There are multiplelimiting algorithms, but, token bucketing. Token bucket. It's our use case quite well. For each user, or each client we'll have to store. How many tokens our kid currently has and when was the last time its bucket was? Refreshed. Using only these two variables, we'll be able to implement our token marketing algorithm. Now the question comes, where to store these two? Ident- user identifiers. If we could go about storing them in each instance of our API gateway, that will be , that will be quite catastrophic for our distributed system. So let's say I'm going to store them in a fast in-memory Redis cluster, which each API gateway instance will refer to for getting a user's, users identifiers, which are how many tokens it currently has and when was its bucket last filled. Now, each time a user makes a request, we'll have to check with Redis get. user's values from Redis. Update them. And after updating, if the current request is allowed, let it go through, otherwise reject it. Now the issue comes. In case of , since this is like a distributed system, this , in case of this distributed lock, we face into the problem of how to read and update these. while handling these contentions. To ensh, to ensure that, we'll use Lua scripting and Redis. to read, update our Redis in one go, and then have our client API gateway decide.

**Q:** What HTTP status code and headers should your system return when rate limits are exceeded?

> We need to return four twenty nine along with. Helpful headers. The headers can be. X rate limit limit, which will. Identify ceiling for that request, limit ceiling for that request, then X. Rate limit remaining, how many. Requests are left in, left in current window. And X rate limit reset. This will be the timestamp when the. Window resets.

**Deep Dives**

**Q:** How do we scale to handle 1M requests/second?

> In production systems. We will. Maybe just say we will use a Redis cluster and build on with a Redis cluster will handle large sharding. And are scaling. But for the question, let's say. We build our sharding and routing logic into our API gateway. We are running multiple shards of. Redis. And whenever a request arrives at API gateway, it checks for all the user identifiers and the rules. And based on that routes. The requests. To one of the Redis shards. And. In between them sits a consistent hashing algorithm. So that for a user, you always land up on the same shard. Otherwise, there'll really a problem.

**Q:** What happens when our rate limiting system fails? Should we pass all traffic or block all traffic, and how do we recover quickly?

> Each Redis cluster will have replicas. Which are. Always on the standby as soon as the primary fails. We will have our replicas take over. Though, due to async replication, between Redis cluster. Some context might be lost. For each client, but that's acceptable in case of failures. And even if we are not able to recover using our replicas, there are like two options. For failed open and failed closed. For failed open, we'll allow all the requests to go through, and for failed closed, we'll allow, we will not allow any requests to go through. Our system might have a payment. Related thing. And the U. The field open thing can call, cause a cascading failure, so. Here we can choose field closed. We'll try getting our system back up. From Redis snapshots. As soon as possible. But till then, all the users will face outage.

**Q:** You chose fail closed for payment related traffic. How would you make sure the gateway switches into that mode quickly and consistently across instances when Redis starts timing out or partially failing?

> We can have. API gateway monitor. Health checks for our Redis cluster. And to rule out temporary failures, we can have, let's say, if five health checks fail, we are doing a health check every five hundred milliseconds, and if five health checks fail in a group, we'll consider our Redis cluster down. And then we can fall back into our failed-close mode.

**Q:** How do we minimize latency overhead?

> At million requests per second. The network latency to get. Redis values can be a concern. So the most important optimization can be a connection pooling technique. Instead of opening and closing connections on every request, we can maintain a pool of connections and then if we get, we can use those connections. As and when needed to get. To reach out to Redis. And when the request finishes, the connection is returned to the connection pool. Geographic distribution provides the biggest latency win. If we deploy our rate limiting infrastructure close to users, that is API gateways and Redis clusters on multiple regions. Then we can have a little bit of latency.

**Q:** You mentioned deploying gateways and Redis clusters in multiple regions. How would you route a client to the right regional limiter while keeping the check fast and avoiding extra hops when traffic patterns shift?

> Shh . Each user will reach out to his nearest API gateway using DNS or our cloud provider. Any custom style. For our Redis, users can be hashed or by DNS. Reach out to their regional Redis. And when traffic shifts. We can either think of shifting the user's state. Into the new Redis. Because otherwise making calls. Cross region calls to get that user's state is not feasible.

**Q:** How do we handle dynamic rule configuration? Where are rules stored and how does our gateway know about them?

> Since the rules are dynamic and if I get faced, we'll need to watch them. We can update our design to add a coordination service. Let's say, for example, zookeeper, all the rules are maintained there. All the API gateway instances watch the rules. path in zookeeper. And whenever a rule is changed, all the API gateway instances are informed in one go.

​