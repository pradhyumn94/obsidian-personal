# HLD Index

Auto-tracked via Dataview — status/category/date come from each note's frontmatter, so this list can't drift out of sync. Add new problem names to the `problems` object below when planning ahead; once you write the note (with `status: Done` in its frontmatter), it checks itself off automatically.

```dataviewjs
const problems = {
	"Fundamentals": ["Load Balancer", "API Gateway", "Distributed Cache", "Distributed Lock", "Rate Limiter", "Logging System", "Metrics Platform"],
	"Storage": ["URL Shortener", "Dropbox", "Google Drive", "Distributed File System"],
	"Communication": ["WhatsApp", "Slack", "Email Service", "Notification Service", "Chat Application"],
	"Social": ["Twitter/X", "Instagram Feed", "Facebook News Feed", "News Aggregator"],
	"Search": ["Search Autocomplete", "Search Engine", "Recommendation System"],
	"Streaming": ["YouTube", "Netflix", "Spotify"],
	"Commerce": ["Amazon", "Shopping Cart", "Payment Gateway", "Order Management"],
	"Mobility": ["Uber", "Ride Matching", "Food Delivery"],
	"Scheduling": ["Distributed Job Scheduler", "Calendar", "Cron Service"],
	"AI Systems": ["AI Agent Platform", "RAG Platform", "Vector Search", "LLM Gateway", "AI Evaluation Platform"],
};

const allPages = dv.pages('"Prep 2026/02 HLD" or "Excalidraw"').array();

let done = 0, total = 0;
for (const [category, items] of Object.entries(problems)) {
	dv.header(2, category);
	const rows = items.map(name => {
		total++;
		const page = allPages.find(p => p.file.name.toLowerCase() === name.toLowerCase());
		const isDone = page && page.status === "Done";
		if (isDone) done++;
		const link = page ? page.file.link : name;
		const meta = isDone && page.date ? ` — ${page.date}` : "";
		const box = isDone ? "✅" : "⬜";
		return `${box} ${link}${meta}`;
	});
	dv.list(rows);
}

dv.paragraph(`**Progress: ${done} / ${total} done**`);
```
