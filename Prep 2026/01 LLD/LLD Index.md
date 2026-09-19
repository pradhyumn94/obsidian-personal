# LLD Index

Auto-tracked via Dataview — status/category/date come from each note's frontmatter, so this list can't drift out of sync. Add new problem names to the `problems` object below when planning ahead; once you write the note (with `status: Done` in its frontmatter), it checks itself off automatically.

```dataviewjs
const problems = {
	"Infrastructure Components": ["LRU Cache", "Rate Limiter", "Task Scheduler", "Logging Framework", "Notification System", "File System", "Circuit Breaker"],
	"Systems & Machines": ["Parking Lot", "Elevator System", "Amazon Locker", "Vending Machine", "ATM Machine", "Library Management System"],
	"Booking & Reservation": ["Movie Ticket Booking System", "Hotel Reservation System", "Airline Reservation System"],
	"Mobility & Delivery": ["Ride Matching Engine", "Food Delivery System", "Parking Spot Allocation"],
	"E-Commerce": ["Shopping Cart", "Order Management System", "Inventory Management System"],
	"Social & Messaging": ["Chat Application", "Social Network Graph", "News Feed"],
	"Games": ["Chess Game", "Tic Tac Toe", "Snake and Ladder", "Card Game (Blackjack)"],
	"AI Systems": ["LLM Prompt Pipeline", "Agent Tool Router", "Conversation Memory Store"],
};

const allPages = dv.pages('"Prep 2026/01 LLD" or "Excalidraw"').array();

let done = 0, total = 0;
for (const [category, items] of Object.entries(problems)) {
	dv.header(2, category);
	const rows = items.map(name => {
		total++;
		const page = allPages.find(p => p.file.name.toLowerCase() === name.toLowerCase()
			|| p.file.name.toLowerCase() === `lld - ${name.toLowerCase()}`);
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
