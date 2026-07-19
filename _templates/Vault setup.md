# Vault setup

## New machine

1. Clone this repo and open the folder as an Obsidian vault.
2. **Settings → Community plugins**: enable, then match [`.obsidian/community-plugins.json`](../.obsidian/community-plugins.json) — currently: Templater, Tasks, Obsidian Git, Dataview, Omnisearch, Settings Search, Local REST API, Linter, Excalidraw, Kanban, Terminal.

## How notes get created

There's no daily-notes or auto-template automation wired up. Templates in this folder are copy-paste (see [`README.md`](README.md)):
1. Open the relevant template file.
2. Select all, copy.
3. Create your new note in the right `Prep 2026/` subfolder, paste, then fill in the placeholders and delete them.

## Structure

Everything prep-related lives under `Prep 2026/`, numbered by track: `01 LLD/`, `02 HLD/`, `03 Engineering Notes/`, `04 Weekly log/`, `05 Interview Notes/`. Each subfolder has its own `README.md` defining what belongs there and the naming convention — read that before adding a note, not this file.

## Weekly log

`Prep 2026/04 Weekly log/` holds one file per week (`Week N (date range).md`), created manually from `Weekly Review Template.md`. There's no Templater startup script generating these — no `Work log/` or `1 Projects/` folder exists in this vault (that was an earlier, since-replaced setup).
