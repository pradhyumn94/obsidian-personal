# Vault setup

## Rhythm

- **Core Daily notes** are off. Weeks are created by **Templater** ([`_templates/Startup.md`](_templates/Startup.md)) when you open Obsidian **Monday–Wednesday** if `Work log/<week-start>.md` is still missing.
- There is no `.obsidian/daily-notes.json` by design (weekly-only workflow).

## New machine

1. Clone this repo and open the folder as an Obsidian vault.
2. **Settings → Community plugins**: enable, then install **Templater**, **Tasks**, **Dataview**, **Obsidian Git**, **Terminal**, **Excalidraw** (or match [`.obsidian/community-plugins.json`](.obsidian/community-plugins.json)).
3. **Templater** startup path is versioned in [`.obsidian/plugins/templater-obsidian/data.json`](.obsidian/plugins/templater-obsidian/data.json) (see `.gitignore` exception). Reload if startup templates do not run.

## Projects and the weekly list

Project notes under `1 Projects/` use YAML **`status`**. The weekly template lists only:

- **`active`** — shown under **Projects** in new week files.

Other values (e.g. **`planning`**, **`paused`**, **`done`**) are **not** listed until you set `status: active` for work you are driving this week.

## Tasks and Open tasks

- In the work log, link tasks to a project note: `- [ ] [[Your project]] …`
- [Open tasks](Open%20tasks.md) lists open tasks from **Work log** and **1 Projects**: **This Week** (current week file) first, then **Everything Else** sorted by note modification time. Headings can show a project wikilink when tasks link to `1 Projects/` (excluding `Archive`).

### If Open tasks looks like a gray code block

1. **Settings → Community plugins**: ensure **Dataview** is installed and enabled.
2. **Settings → Dataview**: turn on **Enable JavaScript Queries** (and inline JS if you use it elsewhere).
3. Use **Reading view** (or Live Preview with Dataview’s live-preview options), not raw **Source mode**, to see the query run.
4. The note must include normal markdown **above** the fenced **dataviewjs** block (e.g. the `# Open tasks` heading). A file that contains **only** a fenced code block and no other markdown can be treated as plain code so Dataview never runs.
