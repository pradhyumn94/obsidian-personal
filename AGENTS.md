_Distilled from prior Claude Code session history on 2026-09-19._

## What this vault is

This is an Obsidian personal prep vault, not a software repo. Treat it like a structured knowledge base for interview prep and review.

## Before editing anything

- Read the local folder's `README.md` or index note first when one exists.
- Prefer small, precise note edits over broad reorgs.
- Do not restructure folders or create new subfolders unless explicitly asked.
- If a file changed concurrently or looks actively edited by the user, re-read and avoid fighting the live change.

## Top-level folder map

- `Prep 2026/` — main interview-prep area.
- `Excalidraw/` — drawing dump; keep Excalidraw files here.
- `_templates/` — copy-paste templates; there is no heavy automation flow.
- `_attachments/` — attachments/supporting assets.

## `Prep 2026/` structure

### `01 LLD/`

- One note per low-level/class-design problem.
- Match existing folder style: plain-English problem names.
- Use `_templates/LLD Template.md` when creating a new note.
- `LLD Index.md` is the tracker for planned/completed LLD problems.

### `02 HLD/`

- One note per high-level/system-design problem.
- Match existing folder style: plain-English system names.
- Use `_templates/HLD Template.md` when creating a new note.
- `Index.md` is the tracker for planned/completed HLD problems.

### `03 Engineering Notes/`

- Permanent reference notes for concepts, not practice-problem writeups.
- Keep this folder flat. Do **not** introduce subfolders preemptively.
- One concept/topic per note.
- Use `_templates/Technical Topic Template.md` for new concept notes.
- HLD/LLD notes should link out to these notes instead of re-explaining concepts inline.
- `Index.md` groups the concept notes by domain.
- `Revision Plan.md` is the spaced-repetition checklist over these notes.

### `04 Interview Notes/`

- `Resume.md` — working resume content.
- `Behavioral Stories.md` — reusable STAR-story bank.
- `Company Notes/` — one note per company.
- Prefer linking from company notes to story-bank entries instead of duplicating stories.

## Engineering Notes style

- Default to concise, interview-focused notes.
- Prefer terse bullets / nested bullets over long prose.
- Keep the core mechanism, trade-offs, common deep dives, and interview angle.
- Drop generic filler and beginner appendices if they do not help recall.
- If asked to condense, tighten language rather than deleting key structure.
- Preserve useful diagrams/flow visuals when summarizing; Mermaid or Excalidraw embeds are fine.

## Linking conventions

- Use Obsidian wikilinks heavily.
- Add bidirectional links when a new concept clearly relates to existing notes.
- `See also:` sections and short related-note lists are common and useful.
- Use full-path wikilinks only when disambiguation is needed.
- If a topic already has a note, merge into the existing note rather than creating a duplicate.

## Excalidraw conventions

- Keep drawings in `Excalidraw/`.
- Topic notes in `Prep 2026/` may embed those drawings with `![[Excalidraw/...]]`.
- Do not move drawings into topic folders unless explicitly asked.

## Recurring workflows

### Adding a new concept note

1. Confirm it belongs in `03 Engineering Notes/` rather than HLD/LLD.
2. Start from `_templates/Technical Topic Template.md`.
3. Write a concise note in the existing terse style.
4. Add cross-links/backlinks to obviously related notes.
5. Add it to `03 Engineering Notes/Index.md` in the right domain section.

### Adding a new HLD or LLD note

1. Use the matching template from `_templates/`.
2. Save it in `01 LLD/` or `02 HLD/`.
3. Update the relevant index checklist.
4. Link to Engineering Notes for reusable concepts instead of duplicating reference material.

### Updating weekly progress

1. Create/update the relevant weekly log from the weekly template.
2. Record concrete completed notes/problems, wins, weak areas, and next steps.
3. Keep `Roadmap.md`, weekly logs, and revision work aligned.
4. If reconstructing a missed week, clearly label it as reconstructed rather than guessed.

## Frequently referenced notes

- `Prep 2026/Roadmap.md`
- `Prep 2026/01 LLD/LLD Index.md`
- `Prep 2026/02 HLD/Index.md`
- `Prep 2026/03 Engineering Notes/Index.md`
- `Prep 2026/03 Engineering Notes/Revision Plan.md`
- `_templates/Vault setup.md`

## Safe defaults

- Prefer updating README/index notes to match reality over reorganizing content.
- Prefer merging and tightening existing notes over spawning near-duplicates.
- Preserve the current Prep-2026 hierarchy.
- When uncertain where something belongs, favor the lightest change and follow existing folder usage.
