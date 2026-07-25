# Backend Roadmap

Progress tracker. Auto-computed: a topic is marked done if its keyword appears anywhere in a note under `03 Engineering Notes/`. No manual checkboxes to maintain — write the note, and it flips to ✅ on next render.

```dataviewjs
const folder = "Prep 2026/03 Engineering Notes";
const topics = {
  "Networking": ["TCP", "UDP", "HTTP", "HTTP2", "HTTP3 - QUIC", "TLS", "gRPC", "WebSockets"],
  "Databases": ["Indexes", "B+ Trees", "MVCC", "Transactions", "Isolation Levels", "Query Optimizer", "Replication", "Partitioning", "Sharding"],
  "Distributed Systems": ["CAP", "Consistency", "Replication", "Leader Election", "Quorum", "Vector Clocks", "Gossip", "Raft", "Paxos", "Event Sourcing", "Saga", "Idempotency"],
  "Messaging": ["Kafka", "RabbitMQ", "Pulsar", "SQS"],
  "Redis": ["Data Structures", "Cluster", "Sentinel", "Pub/Sub", "Streams", "Persistence"],
  "Cloud": ["Docker", "Kubernetes", "Service Mesh", "Helm", "CI/CD"],
};

const files = app.vault.getMarkdownFiles().filter(f => f.path.startsWith(folder));
let corpus = "";
for (const f of files) {
  corpus += (await app.vault.cachedRead(f)).toLowerCase() + "\n";
}

let totalDone = 0, totalTopics = 0;
for (const [section, items] of Object.entries(topics)) {
  dv.header(3, section);
  const rows = items.map(topic => {
    totalTopics++;
    const found = corpus.includes(topic.toLowerCase());
    if (found) totalDone++;
    return [found ? "✅" : "⬜", topic];
  });
  dv.table(["", "Topic"], rows);
}
dv.paragraph(`**Progress: ${totalDone}/${totalTopics}**`);
```
