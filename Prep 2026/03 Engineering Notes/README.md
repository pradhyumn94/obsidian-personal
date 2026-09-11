# Engineering Notes

## What belongs here
Permanent reference notes on core topics, kept flat (no subfolders) in this one directory — spans AI, distributed systems, databases, networking, concurrency, Kafka, Redis, and cloud/infra topics.

Unlike `HLD/`, `LLD/`, and `DSA/`, these aren't tied to a specific practice problem — they're the underlying concepts you draw on when designing systems or answering "how does X work" questions.

## How to use it
1. When you hit a concept worth remembering (in a design session, mock interview, or reading), copy `Templates/Technical Topic Template.md` into this folder.
2. Link out from HLD/LLD notes to these topic notes instead of re-explaining the concept inline (e.g. an HLD note's "Caching" section can link to `Engineering Notes/Caching.md`).
3. If this folder grows large enough that flat browsing gets unwieldy, consider splitting into subfolders by domain then — don't do it preemptively.

## Naming convention
Plain English topic names, one concept per note, title-cased where it reads naturally: `CAP Theorem.md`, `DB Indexing.md`, `Kafka fundamentals.md`. Consistency isn't strictly enforced — favor a name that's easy to search for over rigid casing.
