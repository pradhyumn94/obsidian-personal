### The three properties
- **Consistency**: All nodes see the same data at the same time.
- **Availability**: Every request to a non-failing node receives a response, without the guarantee that it contains the most recent version of the data.
- **Partition Tolerance**: The system continues to operate despite arbitrary message loss or failure of part of the system (i.e., network partitions between nodes).

In distributed systems, **Partition Tolerance** is a given — you actually only get to choose between the other two, based on system requirements.

### Choose Availability
- **Social media**
- **Content platforms** (Netflix)
- **Review sites** (Yelp)

### Choose Consistency
- **Ticket booking systems**
- **E-commerce inventory**
- **Financial systems**

### Levels of consistency
- **Strong consistency**
- **Causal consistency**: Related events appear in the same order to all users.
- **Read-your-own-writes**: Users always see their own updates immediately.
- **Eventual consistency**: The system will become consistent over time but may temporarily have inconsistencies.

### Takeaway
Real-world systems frequently need both availability and consistency — just for different features.
