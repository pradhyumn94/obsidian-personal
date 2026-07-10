---
type: moc
tags: [moc, system-design]
---

# System Design — MOC

Each note is one designed system (template: `System Design`). Draw on [[../Distributed Systems/_MOC Distributed Systems|Distributed Systems]] and [[../Databases/_MOC Databases|Databases]] notes for deep-dive reasoning rather than re-deriving it inline.

## Systems Designed
```dataview
TABLE scale, status
FROM "3 Resources/Engineering/System Design"
WHERE type = "system-design"
SORT file.name ASC
```

## Canonical Problems to Cover (checklist)
- [ ] URL shortener
- [ ] Rate limiter
- [ ] News feed / timeline
- [ ] Chat system
- [ ] Distributed job scheduler
- [ ] Notification system
- [ ] Search autocomplete
- [ ] Video streaming (chunking + CDN)
- [ ] Ride-sharing dispatch
- [ ] Payment processing (idempotency, ledger)
- [ ] Multi-tenant SaaS data isolation
- [ ] LLM inference serving / gateway
- [ ] RAG pipeline at scale
- [ ] Agent orchestration platform
