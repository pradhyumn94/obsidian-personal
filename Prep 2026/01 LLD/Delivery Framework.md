1. Requirements - 5min
	- **Primary capabilities** — What operations must this system support?
	- **Rules and completion** — What conditions define success, failure, or when the system stops or transitions state?
	- **Error handling** — How should the system respond when inputs or actions are invalid?
	- **Scope boundaries** — What areas are in scope (core logic, business rules) and what areas are explicitly _out_ (UI, storage, networking, concurrency, extensibility)?

2. Entities and Relationships (~3 minutes)
	1. Identify entities
		- Nouns in the system
		- If something maintains changing state or enforces rules, it likely deserves to be its own entity.
		- If it’s just information attached to something else, it’s probably just a field on another class.
	2. Define relationships
		- Which entity is the orchestrator — the one driving the main workflow?
		- Which entities own durable state?
		- How do they depend on each other? (has-a, uses, contains)
		- Where should specific rules logically live?

3. Class Design (~10-15 mins)
	- For each entity, you'll answer two questions:
		- **State** - What does this class need to remember to enforce the requirements?
		- **Behavior** - What does this class need to do, in terms of operations or queries

4. Implementation (~10 minutes)
	- **Happy Path** - Walk through the method in a linear way: what inputs it receives, the sequence of steps it performs, the internal calls it makes to other classes, and what it returns or how it changes state. You want your interviewer to see how the system actually moves.
	- **Edge Cases** - After the happy path, enumerate the failure modes: invalid inputs, illegal operations, out-of-range values, calls that violate the current system state—anything that must be rejected or handled gracefully. This is an important step to demonstrate that you're thinking like someone who writes production code, not just toy logic.
	- **Verification** - Walk Through a Specific Scenario

5. Extensibility (~5 minutes, if time and level allow)
