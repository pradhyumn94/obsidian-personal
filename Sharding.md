![[Excalidraw/Sharding|Sharding]]


Partioning
- splitting a large table into smaller pieces inside a single database instance
	**Horizontal partitioning**: Split rows across partitions. 
	**Vertical partitioning**: Split columns across partitions.


How to shard data
	**What to shard by** : The field or column you use to split the data. It defines how the data is grouped.
	**How to distribute** : The rule for assigning those groups to shards. It defines how the data is distributed across machines.


### Choosing Your Shard Key
- High Cardinality
- Even distribution
- Aligns with queries


Sharding strategies
1. Range based 
	1. Pros: simplicity and support for efficient range scans 
	- Shard 1 → User IDs 1–1M
	- Shard 2 → User IDs 1M–2M
	- Shard 3 → User IDs 2M–3M
2. Hash based : use a hash function to evenly distribute records across shards
	1. Pros : even distri
3. 
