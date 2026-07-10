---
type: moc
tags: [moc, dsa]
---

# DSA — MOC

Spaced-repetition tracker: [[DSA Revision]]

## By Pattern
```dataview
TABLE difficulty, status
FROM "3 Resources/Engineering/DSA/Problems"
SORT pattern ASC, difficulty ASC
```

## Due for Review
```dataview
TABLE next_review, srs_interval
FROM "3 Resources/Engineering/DSA/Problems"
WHERE status != "mastered" AND next_review <= date(today)
SORT next_review ASC
```
