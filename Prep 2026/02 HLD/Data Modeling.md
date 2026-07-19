
Database Model Options
	1. Relational
	2. Document
	3. Wide column
	4. Key-val store
	5. Graph database

1. Determine the type of database you'll use
2. List the columns needed to fulfill the functional requirements for each entity
3. Specify primary and foreign keys for each relationship
4. Determine which columns need [[DB Indexing|indexes]] (if any)
5. Determine whether you need to denormalize for performance
6. Consider whether [sharding](https://www.hellointerview.com/learn/system-design/core-concepts/sharding) is necessary. If yes, choose a shard key that matches your main access pattern.

![[Screenshot 2026-07-19 at 9.34.15 AM.png]]