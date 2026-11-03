# LLD Index

## Folder Guide

- One note per object-oriented / class-design problem. Copy [[_templates/LLD Template|LLD Template]] and name it after the problem in plain English.
- Sketch classes and relationships before coding; explain which design patterns you chose and why.
- Link reusable concepts to Engineering Notes rather than duplicating them.

## Design References
- [[Prep 2026/01 LLD/Delivery Framework|Delivery framework]]
- [[OOP Concepts]] · [[Design Principles]] · [[Design Patterns Index|Design Patterns]]
- [[HLD Index]] · [[Engineering Notes Index|Engineering Notes]] · [[Study Log]]

## Practice Tracker

Tracked via Dataview from the LLD notes below. Add new problem names to the `problems` object when planning ahead; `status: Done` in a note's frontmatter checks it off. An unchecked item may be planned or an existing note without Done status. Dates come from frontmatter; drawings stay embedded in the problem notes.

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

const allPages = dv.pages('"Prep 2026/01 LLD"').array();

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
