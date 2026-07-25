# AI Roadmap

Progress tracker. Auto-computed: a topic is marked done if its keyword appears anywhere in a note under `03 Engineering Notes/`. No manual checkboxes to maintain — write the note, and it flips to ✅ on next render.

```dataviewjs
const folder = "Prep 2026/03 Engineering Notes";
const topics = {
  "LLM Basics": ["Tokens", "Tokenization", "Context Window", "Temperature", "Sampling"],
  "Prompt Engineering": ["Zero Shot", "Few Shot", "Chain of Thought", "Structured Output", "Tool Calling"],
  "Embeddings": ["Embeddings", "Similarity Search", "Chunking", "Hybrid Search", "Reranking"],
  "RAG": ["Basic RAG", "Advanced RAG", "Context Compression", "Metadata Filtering"],
  "Agents": ["Agent Loop", "Planning", "Reflection", "Memory", "Multi-Agent", "Human in Loop", "MCP"],
  "Production AI": ["Evaluation", "Guardrails", "Hallucination", "Tracing", "Observability", "Cost Optimization", "Model Routing", "Caching"],
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
  dv.table(["Done", "Topic"], rows);
}
dv.paragraph(`**Progress: ${totalDone}/${totalTopics}**`);
```
