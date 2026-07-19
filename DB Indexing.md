![[Excalidraw/DB Indexing|DB Indexing]]
Read up on 
- [ ] BTrees in detail
- [ ] Red black trees
- [ ] Bloom filters




Types of Indexes

1. B Tree Indexes 
	1. Every node in a B-tree follows strict rules:
		- All leaf nodes must be at the same depth
		- Each node can contain between m/2 and m keys (where m is the order of the tree)
		- A node with k keys must have exactly k+1 children
		- Keys within a node are kept in sorted order
	2. Why Btrees are default choice 
		-  They maintain sorted order, making range queries and ORDER BY operations efficient
		- They're self-balancing, ensuring predictable performance even as data grows
		- They minimize disk I/O by matching their structure to how databases store data
		- They handle both equality searches (email = 'x') and range searches (age > 25) equally well
		- They remain balanced even with random inserts and deletes, avoiding the performance cliffs you might see with simpler tree structures


2. LSM Trees 
		1. Negative impact on reads  - Read has to check at multiple places (memTable, immutable MemTables, SSTable)
			- **Bloom Filters**: Each SSTable has an associated bloom filter - a probabilistic data structure that can quickly tell you if a key is definitely NOT in that file. This lets you skip most SSTables without reading them. If the bloom filter says "maybe", you still need to check, but it eliminates the definite misses.
			- **Sparse Indexes**: Since SSTables are sorted, they maintain sparse indexes that tell you the range of keys in each block. If you're looking for user_id=500 and an SSTable only contains keys 1000-2000, you can skip it entirely.
			- **Compaction Strategies**: Different compaction strategies optimize for different workloads. Size-tiered compaction minimizes write amplification but can lead to more files to check. Leveled compaction maintains fewer files but requires more frequent rewrites.
		2. What happens on write
		 - **Memtable (Memory Component)**: New writes go into an in-memory structure called a memtable, typically implemented as a sorted data structure like a red-black tree or skip list. This is extremely fast since it's all in RAM.
		- **Write-Ahead Log (WAL)**: To ensure durability, every write is also appended to a write-ahead log on disk. This is a sequential append operation, which is much faster than random writes.
		- **Flush to SSTable**: Once the memtable reaches a certain size (often a few megabytes), it's frozen and flushed to disk as an immutable Sorted String Table (SSTable). This is a single sequential write operation that can write megabytes of data at once.
		- **Compaction**: Over time, you accumulate many SSTables on disk. A background process called compaction periodically merges these files, removing duplicates and deleted entries. This keeps the number of files manageable and maintains read performance.
		- 
3. Geospatial Indexes
	 - Think why 2 Btree indexes can't solve 2d data lookup - 
	 1. Geohash 
		 1. Keep splitting every block of data into 4 
		 2. locations that are close to each other usually share similar prefix strings 
		 3. once  our 2D locations are mapped into these ordered strings, we can use a regular B-tree index
		 4. **Negative** - two locations on opposite sides of a street that marks a geohash boundary might not share similar prefixes
	 2. Quad tree
	 3. R tree
		  - quadtree rigidly divides each region into four equal parts regardless of data distribution, R-trees adapt their rectangles to the actual data
		  - rectangles aren't constrained to fixed sizes or positions - they adapt to wherever your data actually clusters
		- R-trees can efficiently handle both points and larger shapes in the same index structure. A single R-tree can index everything from individual restaurant locations to delivery zone polygons and road networks
 4. Inverted Indexes

## Index Optimization Patterns
### Composite Indexes
- When we create a composite index, we're really creating a B-tree where each node's key is a concatenation of our indexed columns
- **Order** - Our index on (user_id, created_at) is great for queries that filter on user_id first, but it's not helpful for queries that only filter on created_at. This follows from how B-trees work - we can only use the index efficiently for prefixes of our column list.
### Covering Indexes
- A covering index is one that includes all the columns needed by your query - not just the columns you're filtering or sorting on.