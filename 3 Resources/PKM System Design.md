---
type: reference
tags: [meta, pkm]
updated: 2026-07-10
---

# PKM System Design — How This Vault Works

Reference doc for the vault's own conventions. Read this when the structure feels unclear, not when doing daily work — the daily workflow should feel automatic, not require re-reading this.

## Why PARA (not Johnny Decimal, not pure tags)

This is its own standalone vault — split out from the Adobe-work Obsidian vault because interview-prep/career content is a different life domain with a different audience (nobody but you needs it mixed into Adobe project notes). It's **PARA at the root, folders for permanent domains, tags for cross-cutting facets**. Reasons:

- PARA's actionability axis (Project = time-bound, Area = ongoing standard, Resource = reference) maps directly onto the two things this vault has to hold at once: a **time-bound interview-prep project** (`1 Projects/Interview Prep 2026`) and a **permanent engineering knowledge base** (`3 Resources/Engineering`). Johnny Decimal is a numbering scheme, not an actionability model — it tells you *where* something goes but not *when it expires*. That distinction is the whole point here: interview prep ends, the knowledge base doesn't.
- Folders answer "where does this permanently live" (one true home per note — no duplication). Tags answer "what facets does this note have right now" (difficulty, company, theme) that cut across folders. See "Folder vs Tags" below.

## Folder Structure

```
1 Projects/
  Interview Prep 2026.md          ← the project note (time-bound, archives when cycle ends)
  Interview Prep/
    Companies/                    ← one note per target company
    Mock Interviews/              ← one note per mock, template: Interview Debrief
    Weekly Reviews/
    Monthly Reviews/

2 Areas/
  Career.md                       ← area note (ongoing, never archives)
  Career/
    Leadership/
    Behavioral Stories/           ← STAR stories, reusable across companies/cycles
    Resume/
    Reading Notes/
    Portfolio Project/
    Career Journal/

3 Resources/
  PKM System Design.md            ← this file
  Engineering/
    _MOC Engineering.md
    DSA/                          ← + Problems/ subfolder, DSA Revision.md (SRS tracker)
    Algorithms/                   ← patterns, one note per pattern
    System Design/
    Distributed Systems/
    Backend Engineering/
    Databases/
    Networking/
    Operating Systems/
    Concurrency/
    Cloud/
    Kubernetes/
    AI Platform Engineering/
      LLMs/
      RAG/
      MCP/
      Agentic Systems/
    Observability/

_templates/                       ← copied over from the Obsidian vault at split-off time; now maintained independently here
```

Everything under `3 Resources/Engineering` and `2 Areas/Career` is **permanent** — it should still be valuable in 5 years regardless of whether these specific interviews happen. Everything under `1 Projects/Interview Prep` is **disposable** — it can be archived to `4 Archive` (create if needed) once the cycle closes, without losing any real knowledge, because the knowledge lives in Resources/Areas and Interview Prep only *links* to it.

## Recommended Plugins (mature, actively maintained only)

| Plugin | Purpose | Notes |
|---|---|---|
| Dataview | Live queries (already used in Home.md, MOCs) | Core to this design — every MOC and dashboard depends on it |
| Templater | Template variables (`tp.date.now`, `tp.file.title`) | Already in use; all new templates use it |
| Spaced Repetition (or manual SRS) | DSA/pattern review scheduling | The `DSA Revision.md` table-based SRS is lighter-weight and already working — only add the plugin if the manual table becomes unwieldy |
| Kanban | Interview pipeline board | See Kanban setup below |
| Excalidraw | System design diagrams | Already installed (`Excalidraw/` folder exists) |
| Tag Wrangler | Bulk rename/merge tags | Useful once tag vocabulary below is in active use |
| Various Complements / Front Matter Title | Autocomplete for tags/frontmatter keys | Reduces tag typos, which matter more once dataview queries depend on exact tag strings |

Deliberately **not** recommending: Smart Connections / AI-linking plugins (adds noise, and the whole point of this design is deliberate linking), Full Calendar (Work log already covers weekly cadence), Zotero integration (no indication of heavy paper-reading volume yet — add later if `Reading Notes/` volume grows).

## Core Settings

- **Default new note location:** per-folder (use "same folder as current file" or configure per-template via Templater's folder targeting) — prevents orphaned notes at vault root.
- **New link format:** shortest path when possible — keeps `[[Note Name]]` links readable; Dataview/MOC queries use full paths only where folders collide (e.g., `Backend Topic` template reused across `Backend Engineering`, `Databases`, `Networking`, etc.).
- **Use Markdown links:** off (keep wikilinks) — consistent with existing vault.
- **Detect all file extensions:** on — needed for the PDF/image attachments already in `_attachments`.
- **Show inline title:** on.

## Tagging Strategy

Tags are for **facets that cut across folders**, never for what a folder already encodes. Rule of thumb: if a dataview query would only ever filter by folder path, don't also tag it.

| Tag | Applies to | Purpose |
|---|---|---|
| `#dsa` | DSA problems/patterns | Cross-folder filter (Problems + Algorithms) |
| `#pattern` | Algorithm pattern notes | Distinguish pattern notes from problem notes |
| `#system-design`, `#distributed-systems`, `#backend`, `#databases`, `#networking`, `#operating-systems`, `#concurrency`, `#cloud`, `#kubernetes`, `#ai-platform`, `#observability` | Domain notes | Redundant with folder today, kept only so notes can be found via tag search from outside their folder (e.g., a System Design note that's really about databases) |
| `#interview-prep` | Anything specific to the current cycle | Lets you filter "everything time-bound" separately from permanent knowledge even though both may live under similar-looking notes |
| `#behavioral`, `#star` | Behavioral stories | Facet, not folder-redundant once stories get cross-linked from company notes |
| `#adr` | Architecture decision records | ADRs get created *inside* whatever project/portfolio folder they document — the tag is what makes them findable as a class |
| `#moc` | Map of Content notes | Lets graph view / search filter out navigation notes from content notes |
| `#reading` | Reading notes | Facet across books/papers/blogs regardless of subject folder |

Avoid: tagging by company name as a global tag (`#google`, `#meta`) — use the `company:` frontmatter field instead so it's structured data for Dataview tables, not a tag explosion.

## Metadata / Frontmatter Standards

Every content note (not MOCs) has:
```yaml
type: <one of: dsa-problem, algorithm-pattern, system-design, distributed-systems-concept,
       backend-topic, ai-engineering-topic, adr, reading-notes, company-prep,
       behavioral-story, interview-debrief, weekly-review, monthly-review>
tags: [...]
status: <draft | active | to-review | mastered | reading | proposed | accepted>
created: YYYY-MM-DD
```
`type` is what Dataview queries filter on (more reliable than tags for structural queries, since it's a single controlled value rather than a list). Domain-specific fields (`pattern`, `difficulty`, `company`, `srs_interval`, `next_review`, etc.) are added per-template — see `_templates/`.

## Naming Conventions

- Note titles: plain English, no ID prefixes (`Consistent Hashing.md`, not `DS-042 Consistent Hashing.md`) — Johnny-Decimal-style IDs are skipped deliberately; folder + Dataview does the organizing, IDs would just be one more thing to keep in sync.
- MOC files: prefix `_MOC ` so they sort to the top of their folder and are visually distinct from content notes (`_MOC DSA.md`).
- One DSA problem per note, named after the problem (`Longest Substring Without Repeating Characters.md`), living in `DSA/Problems/`.
- Company notes named exactly after the company (`Stripe.md`) — this is what `company:` frontmatter and folder-based Dataview queries both key off of.

## Graph View Strategy

- Exclude `_templates`, `.trash`, `_attachments` from graph view (Settings → Graph → filters) — pure noise.
- Color groups by `type:` frontmatter (Distributed Systems, System Design, AI Platform, DSA, Interview Prep as separate colors) so the graph visually shows the "permanent knowledge core surrounded by a time-bound interview-prep shell" structure this design intends.
- Expect (and want) System Design and Distributed Systems notes to be the densest hub — that's the correct shape for a backend-heavy staff-track vault.

## Linking Strategy

- **System Design notes link out, never duplicate.** A system design's "Storage Choice" section links to a `Databases` note instead of re-explaining B-trees inline. If you notice yourself re-explaining a concept inline, that's the signal to extract it into its own Resource note and link instead.
- **Company notes link to Behavioral Stories, not the reverse.** Stories are reusable across companies/cycles; keep the coupling one-directional so a story doesn't accumulate a company-specific rewrite.
- **Interview Debriefs link to the gap**, not just describe it — a debrief that exposes a weak spot in, say, consistent hashing should link `[[Consistent Hashing]]` so the Monthly Review's "what's not working" section can trace back to a concrete note to restudy.
- **Weekly/Monthly Reviews are the only notes allowed to "roll up"** — they're the place where scattered daily study is allowed to become prose; every other note type stays atomic (one concept/problem/story per note).

## Daily Workflow (weekday, 2-3h)

1. Open `Home.md` → check Open Tasks + This Week's work log entry.
2. Pick from `DSA Revision.md` due list (~30-45 min, 2-3 problems) → new note per problem in `DSA/Problems/` from `DSA Problem` template, linking to the relevant `Algorithm Pattern` note.
3. One deep-dive block (~1-1.5h): either advance a `System Design` draft, add a `Backend Topic`/`Distributed Systems Concept` note, or read + log a `Reading Notes` entry.
4. Log what happened under today's `Work log` entry, linking to `[[Interview Prep 2026]]` or the specific note touched.

## Weekly Workflow (weekend, 6-8h/day × 2)

1. Saturday: 1-2 full system design reps (45-60 min each, timed) using the `System Design` template; one AI Platform Engineering deep-dive.
2. Sunday: mock interview (once available, ~month 3+) → `Interview Debrief` note; behavioral story drafting/refinement.
3. End of weekend: fill out `Weekly Review` (new note in `Interview Prep/Weekly Reviews/`), pulling honestly from what got logged all week rather than reconstructing from memory.

## Monthly Workflow

1. First weekend of the month: `Monthly Review` note in `Interview Prep/Monthly Reviews/`.
2. Re-score confidence table against last month's — this is the actual progress signal, more reliable than task-completion counts.
3. Adjust `Interview Prep 2026` milestones table if reality has diverged from plan (it will).
4. Prune/merge any DSA patterns that have reached `mastered` status out of the active review rotation.

## Spaced Repetition Workflow

Keep the existing lightweight table-based SRS in `DSA Revision.md` (Day 1 → 3 → 7 → 14 → 30 → 60 → 120) rather than adopting a plugin — it's already working, and a plugin adds review-queue overhead for a domain (algorithm patterns, ~25 items) small enough that a manual table stays legible. Revisit only if pattern count grows past ~40 or if per-problem (not per-pattern) spaced repetition becomes desirable — at that point the `next_review`/`srs_interval` frontmatter fields already on the `DSA Problem` template are ready to feed a plugin without rework.

## Kanban Setup (Interview Pipeline)

Create `1 Projects/Interview Prep/Pipeline.md` using the Kanban plugin with columns:

`Researching → Applied → Phone Screen → Technical Rounds → Onsite/Virtual Onsite → Offer/Rejected`

Each card is a link to the company's `Company Preparation` note (`[[Stripe]]`), so moving a card is the only "status" action needed — the company note's `status`/`target_stage` frontmatter can stay in sync manually or be treated as secondary. Keep one board for the whole cycle, not one per company.

## Task Management Workflow

- Tasks live inline in `Work log` (daily) and `Interview Prep 2026` (milestones) — not scattered across content notes. Content notes (DSA problems, System Design, etc.) don't carry checkboxes; they carry `status:` frontmatter instead, which Dataview reads structurally.
- `Interview Debrief` notes are the one exception: their "Follow-up Actions" checklist is allowed, because a debrief's whole purpose is generating next actions.

## Progress Tracking Dashboards

Beyond `Home.md`'s existing Open Tasks view, add to `Home.md` or a dedicated `Interview Prep 2026` dashboard section:
- DSA due-for-review count (query already in `DSA/_MOC DSA.md`)
- Company pipeline status table (query already in `Companies/_MOC Companies.md`)
- Mock interview outcome trend (query already in `Mock Interviews/_MOC Mock Interviews.md`)
- Weekly Review streak — a simple `dv.pages` count of `Weekly Reviews` notes created, to make consistency visible

## Dataview Queries — where they live

Every MOC file above already embeds the relevant query (list/table filtered by `type`, folder, or frontmatter field) — that's the intended pattern: **MOCs are the dashboards**, `Home.md` stays focused on daily task triage rather than becoming a monolith. Add new queries to the relevant MOC, not to `Home.md`, unless it's genuinely a daily-relevant view.

## Folder vs Tags — the actual rule

- **One note = one folder**, always, no exceptions. Folder = the note's permanent home = what backup/archive/move operations act on.
- **Tags = facets that would otherwise require duplicating the note** (a System Design note that's also relevant to a specific company gets `company:` in frontmatter, not moved into that company's folder).
- If you ever want to ask "show me everything about X regardless of where it lives," that's a tag or a Dataview query on frontmatter — never a reason to move a note between folders.
