
## What it is
Single entry point that sits in front of backend services and routes client requests to the appropriate one. Analogy: hotel front desk — clients don't need to know the internal structure of your microservices, they just talk to the gateway. Emerged alongside microservices: once a monolith splits into many services, clients need one stable place to talk to instead of tracking down each service directly.

**Core responsibility = request routing.** Everything else (auth, rate limiting, caching, etc.) is secondary middleware bolted on top. Common interview mistake: candidates list all the middleware but forget to say routing is the actual reason it exists.

## Request Lifecycle
1. **Validation** — reject malformed requests early (bad URL, missing headers/API key, malformed body) before they waste backend resources.
2. **Middleware** — auth (JWT), rate limiting, SSL termination, logging, CORS, IP allow/deny lists, request-size validation, API versioning, throttling, service discovery integration. Most interview-relevant: **auth, rate limiting, IP allow/deny**.
3. **Routing** — routing table maps path + method + query params + headers → backend service (e.g., `/users/*` → user-service).
4. **Backend communication** — if a backend uses a different protocol internally (e.g., gRPC), the gateway translates so clients don't need to know/care.
5. **Response transformation** — converts internal response format (e.g., gRPC) back into the client-facing format (e.g., JSON over HTTP).
6. **Caching (optional)** — cache responses that are non-user-specific and don't change often. Full-response caching, partial caching, or TTL/event-based invalidation; backed by in-memory or a distributed cache like [[Redis]].

## Scaling
- **Horizontal scaling** — gateways are stateless, so just add instances behind a load balancer. Two distinct load-balancing layers worth knowing (but usually collapsible to one box in an interview):
  - Client → Gateway: handled by a dedicated LB (e.g., AWS ELB, NGINX).
  - Gateway → Service: gateway itself can load-balance across backend instances.
- **Global distribution** — deploy gateway instances per-region (like a CDN) + GeoDNS routing to the nearest one; keep routing rules/policies synced across regions.

## Popular Implementations
- **Managed**: AWS API Gateway (Lambda integration, usage plans, CloudWatch), Azure API Management (OAuth/OIDC, policy-based), Google Cloud Endpoints (gRPC-heavy, GCP-native). Easiest, most expensive.
- **Open source**: Kong (NGINX-based, plugin ecosystem, service-mesh capable), Tyk (native GraphQL, multi-DC), Express Gateway (Node.js, lightweight).

## When to Use
✅ Microservices architecture — clients otherwise need to know about and couple to every individual service.

❌ Simple monolith / single client type — adds unneeded complexity and a hop.

## Interview Angle
- This is a **thin, low-detail component** — don't over-invest time here. State it, justify it briefly, move on.
- Suggested line: *"I'll add an API Gateway to handle routing and basic middleware (auth, rate limiting)."*
- Bigger risk in interviews is over-explaining the gateway at the expense of the actual system design.

## Related Topics
- [[Networking 101]]
- [[GraphQL Foundations]]
- [[Caching]]
- [[Redis]]
