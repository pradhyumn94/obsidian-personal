# Real-Time Updates (System Design)

## The Problem
Real-time systems (chat, collaborative docs, live dashboards) need servers to **push** updates to clients — standard HTTP request/response can't do this. Two distinct sub-problems ("hops"):

```mermaid
flowchart LR
    U[Update Source] -->|Hop 2: how does server get triggered?| S[Server]
    S -->|Hop 1: how do updates reach client?| C[Client]
```

---

## Hop 1: Client ↔ Server Protocols

### Networking primer
- **L3 (IP)**: routing/addressing, best-effort delivery, no ordering/loss guarantees.
- **L4 (TCP/UDP)**: TCP = connection-oriented, ordered, reliable, costly to set up. UDP = connectionless, no guarantees.
- **L7**: HTTP, WebSocket, WebRTC — built on TCP.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: SYN
    S->>C: SYN-ACK
    C->>S: ACK
    Note over C,S: TCP established
    C->>S: HTTP GET
    S->>C: HTTP Response
    C->>S: FIN / ACK
    S->>C: FIN / ACK
```
Key takeaways: each round trip adds latency; the TCP connection is state both sides must maintain (keep-alive avoids re-handshaking).

### Load balancers
- **L4 (e.g. AWS NLB)**: routes by IP/port only, preserves one persistent TCP connection client→server. Best for WebSockets.
- **L7 (e.g. AWS ALB)**: terminates and re-opens connections, can route on URL/headers/cookies. More flexible, better for HTTP-style traffic (long polling), spottier WebSocket support.

### Options (from least to most capable)

**1. Simple Polling** — client hits server on a fixed interval.
```mermaid
sequenceDiagram
    loop every 2s
        Client->>Server: GET /updates
        Server-->>Client: current state
    end
```
- ✅ dead simple, stateless, no special infra
- ❌ latency ≈ poll interval, wasted requests, DB load
- Use when: not latency-sensitive, or update window is short.

**2. Long Polling** — server holds the request open until data is available, then client immediately re-requests.
```mermaid
sequenceDiagram
    Client->>Server: GET /updates
    Note over Server: holds request open
    Server-->>Client: data ready
    Client->>Server: GET /updates (new request)
```
- ✅ builds on plain HTTP, stateless server-side
- ❌ each update requires a fresh round trip → bursty updates arrive with compounding latency; browsers cap concurrent connections/domain
- Use when: infrequent updates or "let me know when this long job finishes" (e.g. payment status).

**3. SSE (Server-Sent Events)** — one HTTP response, sent as an open `chunked` stream; server pushes chunks as they occur.
```mermaid
sequenceDiagram
    Client->>Server: GET /updates (EventSource)
    Note over Server: connection stays open
    Server-->>Client: event 1
    Server-->>Client: event 2
    Server-->>Client: event 3
```
- ✅ native browser support (`EventSource`), auto-reconnect w/ `Last-Event-ID`, efficient (no reconnect per message)
- ❌ one-way only; some proxies/LBs buffer streamed responses (breaks silently); connection caps per domain
- Use when: one-way, moderate-frequency updates — live dashboards, AI token streaming.

**4. WebSockets** — HTTP "Upgrade" to a persistent, full-duplex TCP-like channel.
```mermaid
sequenceDiagram
    Client->>Server: HTTP Upgrade request
    Server-->>Client: 101 Switching Protocols
    Note over Client,Server: full-duplex channel open
    Client->>Server: message
    Server->>Client: message
```
- ✅ full-duplex, low overhead, high frequency
- ❌ needs WS-aware infra end-to-end, stateful connections complicate LB/scaling/deploys, must handle reconnection
- Common pattern to tame statefulness: terminate WS in a dedicated **WebSocket service**, keep everything downstream stateless.
```mermaid
flowchart LR
    User --> L4[L4 Load Balancer] --> WS[WebSocket Service] --> Driver[Driver Service]
```
- Use when: high-frequency, bidirectional (chat, collab cursors). Don't reach for it just for server→client push — SSE is usually enough.

**5. WebRTC** — peer-to-peer, browser-to-browser. Signaling server matches peers; STUN does NAT hole-punching; TURN relays as fallback.
```mermaid
flowchart TB
    A[Client A] <-->|1. connect, learn peers| Sig[Signaling Server]
    B[Client B] <-->|1. connect, learn peers| Sig
    A -.->|2. STUN: get public addr| Stun[STUN Server]
    B -.->|2. STUN: get public addr| Stun
    A ===|4. direct P2P data/media| B
    A -.->|3. fallback relay| Turn[TURN Server]
    Turn -.-> B
```
- ✅ direct P2P, lowest latency, offloads server cost, native A/V
- ❌ most complex, needs signaling infra, NAT issues, connection setup delay
- Use when: audio/video calls, or P2P collaboration (e.g. Canva pointers, CRDTs) to cut server load.

### Decision flow
```mermaid
flowchart TD
    A{Latency sensitive?} -->|No| P[Simple Polling]
    A -->|Yes| B{Frequent, bi-directional?}
    B -->|No| SSE[SSE]
    B -->|Yes| C{Peer-to-peer / audio-video?}
    C -->|No| WS[WebSocket]
    C -->|Yes| RTC[WebRTC]
```

---

## Hop 2: Getting the Server Triggered

### Pull via Polling
Updates land in a DB; clients poll it. Decouples producer from consumer, but reintroduces latency.
```mermaid
flowchart LR
    Src[Update Source] -->|write| DB[(DB)]
    Client -->|poll| Server --> DB
```
⚠️ Watch read QPS: 1M clients × poll/10s = 100k TPS.

### Push via Consistent Hashing
Persistent-connection servers (WS/SSE) each "own" a subset of users. A coordination service (ZooKeeper/etcd) maps users→servers via a hash ring so scaling only reshuffles a small slice of connections (vs. modulo hashing, which reshuffles almost everyone).
```mermaid
flowchart LR
    ZK[ZooKeeper/etcd] --- S1[Server 1]
    ZK --- S2[Server 2]
    Upd[Update Server] -->|hash user → server| S2
    S2 -->|lookup connection| UserC[User C]
```
- ✅ predictable placement, minimal churn on scale, good for heavy per-connection state
- ❌ complex, needs coordination service, connection state lost if a node dies
- Use when: connections carry expensive state (e.g. a loaded doc in Google Docs).

### Push via Pub/Sub
Lightweight "endpoint servers" just hold client connections and subscribe to topics on a Pub/Sub layer (Kafka/Redis). Any client can land on any endpoint server — state lives in Pub/Sub, not the servers.
```mermaid
flowchart LR
    UserA & UserB & UserC --> E1[Endpoint Server 1] & E2[Endpoint Server 2]
    E1 & E2 <-->|subscribe / publish| PS[(Pub/Sub: Redis/Kafka)]
    UpdSrc[Update Source] -->|publish to topic| PS
```
- ✅ endpoint servers stay stateless/interchangeable, easy "least connections" LB, efficient broadcast
- ❌ no visibility into connect/disconnect, Pub/Sub is a SPOF/bottleneck (mitigate w/ Redis Cluster sharding), extra hop latency
- Use when: broadcasting to many clients without needing much per-connection state — default choice for most systems.

---

## Using This in Interviews
- **Proactively** call out real-time needs early: "messages need instant delivery → WebSockets" / "character edits need sub-second propagation."
- **Default to the simplest thing that works.** Polling avoids both hops entirely — great if latency isn't core to the ask.
- Escalation path: Simple Polling → Long Polling → SSE → WebSocket → WebRTC. Don't skip ahead without justification.

### Common scenarios
| Scenario | Hop 1 | Hop 2 |
|---|---|---|
| Chat | WebSocket | Pub/Sub |
| Live comments (celebrity fan-out) | SSE/WS + hierarchical aggregation | Pub/Sub + batching |
| Collaborative doc editing | WebSocket (+ CRDT/OT for conflicts) | Consistent hashing (doc = expensive state) |
| Live dashboards/analytics | SSE | Pull polling or Pub/Sub |
| Gaming | WebRTC (P2P) + WebSocket (coordination) | Pub/Sub |

### Common deep-dive questions
- **Reconnection/failure handling**: heartbeats to detect "zombie" connections; track last-seen sequence/message ID per client so reconnects can backfill (Redis Streams are a common tool).
- **Celebrity fan-out (millions of followers)**: don't fan out writes to every feed — cache once, distribute through regional/hierarchical broadcast layers.
```mermaid
flowchart TB
    Root[Root Processor] --> WP1[Write Processor 1] & WP2[Write Processor 2] & WP3[Write Processor 3]
    WP1 & WP2 --> BN1[Broadcast Node 1]
    WP2 & WP3 --> BN2[Broadcast Node 2]
    BN1 --> UsersA[User A, B...]
    BN2 --> UsersB[User C, D...]
```
- **Message ordering across distributed servers**: vector clocks/logical timestamps in theory; in practice, for most product-design interviews, funnel related messages through a single server/partition and stamp a total order there.

## Bottom line
Two hops, five client protocols, three propagation patterns. Start simple (polling), escalate only when the problem genuinely demands lower latency or bidirectionality — and always state the trade-off out loud.
