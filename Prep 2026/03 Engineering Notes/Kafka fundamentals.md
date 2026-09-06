
![[Kafka]] 

See also: [[Redis]] (Pub/Sub and streams cover similar ground at lower throughput)

Motivation:
	Problem 1 : Too many events on the queue
	Solution : Scale horizontally (more servers)
	
	Problem 2 : Events are not processed in order
	Solution: Partition by topic (game)
	
	Problem 3 : Consumers can't keep up with rate of events
	Solution : Consumer group
	
	Problem 4:  Want to keep each sport separate
	Solution: Topics


Terminlogy:
	**Brokers** : The  server(physical/virtual) that hold the queue
	**Partition**: The "queue".  An ordered immutable sequence of messages that we append to like log file. Each broker can have multiple partitions.
	**Topics** : Logical group of partitions. You publish to and consume from Topics in Kafka
	**Producers** : Write messages/events to Topics
	**Consumers** : Read messages/events from Topics
	  


