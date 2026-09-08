
## 1. The Core Problem & Leader Election

In a distributed system, nodes communicate over unreliable networks. Allowing multiple nodes to write simultaneously leads to race conditions, conflicting states, and data corruption. To solve this, distributed systems enforce the **single-writer principle**: only one designated node (the leader) coordinates writes, sequences operations, and drives state changes. **Leader election** is the protocol nodes use to agree on who holds this role.

## 2. What is a Quorum?

A **quorum** is the minimum number of participating nodes that must agree on an operation or election result to be considered valid. In most crash-fault-tolerant systems, a quorum is defined as a strict majority:

$$Q = \lfloor \frac{N}{2} \rfloor + 1$$
### Why Quorums Matter

- **Preventing Split-Brain:** If a network partition splits a 5-node cluster into a group of 2 and a group of 3, a quorum requirement of 3 ensures only the majority partition can elect a leader and process writes, preventing two separate leaders from mutating state independently.
- **Overlap Guarantee:** Any two quorums intersect by at least one node, ensuring history and committed data are never lost across leader transitions.

## 3. How Leader Election and Quorum Work Together (Raft Model)

1. **States:** Nodes act as Followers, Candidates, or Leaders.
2. **Heartbeats & Timeouts:** Followers listen for heartbeats. If a heartbeat times out, a follower increments its term, votes for itself, and becomes a **Candidate**.
3. **Requesting Votes:** The candidate broadcasts a `RequestVote` RPC.
4. **Quorum Check:** Nodes grant votes only if the candidate's term is current, they haven't voted yet, and the candidate's log is up-to-date.
5. **Winning:** Receiving votes from a **quorum** promotes the candidate to **Leader**.

## 4. Senior-Level Nuances (Advanced)

- **Pre-Vote Phase:** Prevents isolated nodes with expired terms from causing global election storms when they reconnect.
- **Zombie Leaders & Read Quorums:** Isolated leaders can serve stale data. Systems use **ReadIndex** or **Leader Leases** to verify leadership before reads.
- **Fencing Tokens:** Monotonically increasing version numbers reject writes from deposed, slow-moving leaders.
- **Dynamic Membership Changes:** Safe cluster scaling requires complex protocols (like Joint Consensus) to prevent overlapping majorities.
- **Garbage Collection & Stalls:** GC pauses can cause a leader to stop sending heartbeats; kernel watchdogs or health checkers force-kill stalled nodes to prevent flapping.

## 5. Raft Log Replication

1. **Client Request:** A write request is sent to the leader.
2. **Local Append & Broadcast:** The leader appends the entry to its local log and broadcasts an `AppendEntries` RPC in parallel to all followers.
3. **Quorum Acknowledgment:** Followers write the entry to disk and acknowledge it. Once a **quorum** responds, the entry is **committed**.
4. **Commit & Respond:** The leader applies the entry to its state machine and returns success to the client.
5. **Handling Lags & Crashes:** Slow followers catch up automatically via log backtracking. If a leader crashes with uncommitted entries, the new leader's log takes precedence, safely overwriting the uncommitted history.
    

## 6. Log Compaction (Snapshotting)

To prevent logs from growing infinitely, nodes periodically take a snapshot of their current state machine, store it to disk, and truncate all historical log entries up to that index.