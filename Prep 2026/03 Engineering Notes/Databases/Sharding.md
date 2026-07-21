![[Excalidraw/Sharding|Sharding]]

### Partitioning
- splitting a large table into smaller pieces inside a single database instance
- **Horizontal partitioning**: Split rows across partitions.
- **Vertical partitioning**: Split columns across partitions.

### How to shard data
- **What to shard by**: The field or column you use to split the data. It defines how the data is grouped.
- **How to distribute**: The rule for assigning those groups to shards. It defines how the data is distributed across machines.

### Choosing Your Shard Key
- High Cardinality
- Even distribution
- Aligns with queries

### Sharding strategies
1. **Range based**
	- Pros: simplicity and support for efficient range scans
	- Shard 1 → User IDs 1–1M
	- Shard 2 → User IDs 1M–2M
	- Shard 3 → User IDs 2M–3M
2. **Hash based**: use a hash function to evenly distribute records across shards
	- Pros: even distribution
	- Cons: data movement on adding/removing shards. Consistent hashing solves this.
3. **Directory based sharding**
	- user_to_shard
		- User 15 → Shard 1
		- User 87 → Shard 4
		- User 204 → Shard 2
	- Pros: flexibility, celebrity user to dedicated partition, update table → rebalance, can implement complex sharding logic
	- Cons: Extra lookup on every query

### Challenges

#### Hot Spots and Load Imbalance
- Celebrity problem
	- Hash-based distribution doesn't help here because the issue isn't the distribution strategy, it's that some keys are inherently more active than others
	- Time-based sharding creates a different kind of hot spot. If you shard by creation date, all new writes go to the most recent shard
	- **Handling**
		- Isolate hot keys to dedicated shards
		- Use compound shard keys: hash(user_Id+date)
		- Dynamic Shard Splitting: Some DBs automatically split shard when it gets too hot. MongoDB's balancer: split and migrate range-based chunks (including when using a hashed shard key) to maintain balance

#### Cross shard operations
- can't eliminate cross-shard queries entirely
- **Cache the results**
	- first query is expensive, but the next thousand requests hit the cache instead of querying "n" shards
	- works well for queries that don't need real-time accuracy
- **Denormalize to keep related data together**
	- duplicate some data to make querying easier
	- updates more complex
- **Accept hit for rare queries**

#### Maintaining consistency
- **Design to avoid cross-shard transactions**
- **Use sagas for multi-shard operations**: Break the operation into a sequence of independent steps, each with a compensating action
- **Accept eventual consistency**
