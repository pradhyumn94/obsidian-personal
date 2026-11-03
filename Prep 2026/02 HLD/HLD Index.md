# HLD Index

## Folder Guide

- One note per system design problem. Copy [[_templates/HLD Template|HLD Template]] and use a plain-English system name, without numbering or prefixes.
- Allow 45-60 minutes for a full practice session; fill in the note as you work.
- Keep revision takeaways for pre-interview review. Link reusable concepts to Engineering Notes and embed drawings from `Excalidraw/`.

## Design References
- [[Prep 2026/02 HLD/Delivery framework|Delivery framework]]
- [[API design]] · [[Data Modeling]]
- [[Engineering Notes Index|Engineering Notes]] · [[LLD Index]] · [[Study Log]]

## Practice Tracker

Tracked via Dataview from the HLD notes below. Add new problem names to the `problems` object when planning ahead; `status: Done` in a note's frontmatter checks it off. An unchecked item may be planned or an existing note without Done status. Dates come from frontmatter; drawings stay embedded in the problem notes.

```dataviewjs
const problems = {
	"Fundamentals": ["Load Balancer", "API Gateway", "Distributed Cache", "Distributed Lock", "Distributed Rate Limiter", "Logging System", "Metrics Platform"],
	"Storage": ["URL Shortener", "Dropbox", "Google Drive", "Distributed File System"],
	"Communication": ["WhatsApp", "Slack", "Email Service", "Notification Service", "Chat Application"],
	"Social": ["Twitter/X", "Instagram", "Facebook News Feed", "News Aggregator"],
	"Search": ["Search Autocomplete", "Search Engine", "Recommendation System", "Yelp"],
	"Streaming": ["YouTube", "Netflix", "Spotify"],
	"Commerce": ["Amazon", "Shopping Cart", "Payment Gateway", "Order Management", "Ticket Master"],
	"Mobility": ["Uber", "Ride Matching", "Food Delivery", "Local delivery service"],
	"Scheduling": ["Distributed Job Scheduler", "Calendar", "Cron Service"],
	"AI Systems": ["AI Agent Platform", "RAG Platform", "Vector Search", "LLM Gateway", "AI Evaluation Platform"],
};

const allPages = dv.pages('"Prep 2026/02 HLD"').array();

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
