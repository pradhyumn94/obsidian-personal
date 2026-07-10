# Engineering Notes

## What belongs here
Permanent reference notes on core topics, organized into subfolders by domain:
`AI/`, `Distributed Systems/`, `Databases/`, `Networking/`, `Concurrency/`, `Kafka/`, `Redis/`, `Cloud/`.

Unlike `HLD/`, `LLD/`, and `DSA/`, these aren't tied to a specific practice problem — they're the underlying concepts you draw on when designing systems or answering "how does X work" questions.

## How to use it
1. When you hit a concept worth remembering (in a design session, mock interview, or reading), copy `Templates/Technical Topic Template.md` into the relevant subfolder.
2. Link out from HLD/LLD notes to these topic notes instead of re-explaining the concept inline (e.g. an HLD note's "Caching" section can link to `Engineering Notes/Databases/Cache Invalidation Strategies.md`).
3. Add new subfolders only if a domain genuinely doesn't fit the existing eight — don't create one-off folders for a single note.

## Naming convention
Plain English topic names, one concept per note: `Consistent Hashing.md`, `Cache Invalidation Strategies.md`, `Consumer Groups.md`.
