
## What is ZooKeeper?

**Distributed coordination service** for maintaining small, strongly consistent pieces of shared state.

**Use for:** leader election, distributed locks, service membership, configuration, failure detection.

> **ZooKeeper = control plane, not data plane.**

---

## Architecture & Quorum

```text
        ZooKeeper Ensemble
          Leader
         /      \
   Follower    Follower

3 nodes → quorum = 2 → tolerate 1 failure
5 nodes → quorum = 3 → tolerate 2 failures
```

- One **leader**, multiple followers.
- Writes go through the leader and are replicated.
- **Majority quorum** required for safe progress.
- Lose quorum → **no writes** → avoids split-brain.
- Consensus protocol: **ZAB (ZooKeeper Atomic Broadcast)**.
- **zxid** provides transaction ordering.

### Reads vs Writes

```text
Read  → any server → in-memory copy   (fast, may be stale)
Write → leader only → ZAB commit → quorum ack → durable

Optimized for ~10:1 read:write
Need freshest read? → call sync() before get → forces catch-up with leader
```

### ZAB internal leader election (vs app-level election)

```text
New ZK leader = highest zxid (most up-to-date history)
                tie → highest server ID

App-level election = lowest sequential znode wins
→ opposite rule, don't confuse the two
```

### Storage / durability

```text
Write → transaction log (fsync, perf-critical, dedicated disk)
      → periodic snapshot of in-memory state

Restart → load latest snapshot → replay log → recovered state
```

> Avoid JVM heap swapping — writes are ordered, so one slow request stalls the whole queue.

---

## Znodes

Hierarchical namespace:

```text
/services/payment/worker-1
/config/payment/timeout
/locks/payment
```

### Persistent

Survives client/session failure.

### Ephemeral

Exists only while the client's **session** is alive.

```text
Worker → /services/worker-1 (ephemeral)

Session expires
      ↓
znode deleted
```

**Use ephemeral nodes for liveness/membership.**

---

## Sessions

Client maintains a session using heartbeats.

```text
Network loss
   ↓
Reconnect before timeout → session survives

Timeout exceeded
   ↓
Session expires → ephemeral nodes deleted
```

> Network disconnection ≠ immediate session expiration.

---

## Watches

Client registers a watch on a znode.

```text
/config/payment
      ↓
    watch
      ↓
   changed
      ↓
client notified
```

- Generally **one-shot** → re-register after event.
    
- Avoid everyone watching the same node → **thundering herd**.
    

---

## Sequential Znodes

ZooKeeper can create ordered nodes:

```text
/lock/client-000001
/lock/client-000002
/lock/client-000003
```

Used for **leader election and distributed locks**.

### Leader Election

Smallest sequence number wins:

```text
000001 → Leader
000002 → watches 000001
000003 → watches 000002
```

If `000001` disappears → `000002` becomes leader.

**Watch predecessor, not the leader**, to avoid thundering herd.

---

## Distributed Lock

```text
/lock/000001  ← owner
/lock/000002
/lock/000003
```

Smallest sequence owns the lock; each waiter watches its predecessor.

### Important failure case

```text
A owns lock
 ↓
A pauses / loses network
 ↓
session expires
 ↓
B acquires lock
 ↓
A wakes up
```

A may still believe it owns the lock.

> **Lock ≠ protection from stale owners.**

Use **fencing tokens** for dangerous downstream operations:

```text
A → token 10
B → token 11

Storage rejects stale token 10
```

---

## Network Partition

```text
Z1 ── X ── Z2 + Z3
```

If `Z2 + Z3` have quorum:

```text
→ elect new leader
→ continue
```

Isolated old leader cannot safely commit.

If **no quorum**:

```text
→ stop writes
```

This prevents conflicting histories / split-brain.

---

## When to Use / Not Use

|Use ZooKeeper|Don't use ZooKeeper|
|---|---|
|Leader election|Application data|
|Distributed locks|Large datasets|
|Membership|High-volume data serving|
|Configuration|Caching|
|Coordination|Event streaming|

### Modern ecosystem
- **Kafka:** historically used ZooKeeper → modern Kafka uses **KRaft**
- **Kubernetes:** uses **etcd**
- Still core to Apache ecosystem: HBase, Hadoop, SolrCloud, Storm, Pulsar; ClickHouse uses it for replication.

### Alternatives

```text
etcd    → cloud-native, powers K8s, HTTP/gRPC, small high-read datasets
Consul  → + service discovery, health checks, network automation
Cloud   → AWS Parameter Store/CloudMap, Azure App Config, managed MSK ZooKeeper
```

### Limitations

```text
Hot spotting     → many watchers on one znode → notification storm
Write cost       → every write: leader + quorum fsync → low write throughput
Capacity         → znodes <1MB, full dataset must fit in memory
Ops complexity   → "simple to use, complex to operate"
```

### When it shines in interviews

```text
Smart routing      → coordinator maps chat-rooms/streams → servers (colocate same-room users)
Infra design       → broker registration, partition leader election, rebalancing,
                      failure detection via ephemeral nodes (pre-KRaft Kafka)
Hierarchical locks → nested lock trees (dir + files), deadlock-safe ordering
                      → ZK > Redis for correctness-critical, long-lived locks
```
---

## Core Mental Model

```text
ZooKeeper
   │
   ├── Ensemble → Leader + Followers
   ├── Quorum → Majority for writes
   ├── ZAB → Ordered replicated state
   ├── Sessions → Client liveness
   ├── Ephemeral znodes → Membership
   ├── Sequential znodes → Election / Locks
   └── Watches → Change notification
```

> **ZooKeeper = strongly consistent coordination + quorum + sessions + ephemeral/sequential znodes + watches.**