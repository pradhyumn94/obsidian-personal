
![[Consistent Hashing.excalidraw]]



### Addressing Hot Spots

Consistent hashing doesn't solve this on its own since it distributes keys evenly, not traffic.
- **Read replicas**: Replicate popular keys across multiple nodes and load-balance reads among them.
- **Key-space salting**: Append a random suffix to hot keys (e.g., taylor-swift-{0..9}) so they hash to different nodes. Reads then scatter across those nodes and get aggregated.
- **Adaptive rebalancing**: Monitor traffic in real-time and move specific key ranges off overloaded nodes. This is operationally complex but some systems (like DynamoDB) do it automatically.