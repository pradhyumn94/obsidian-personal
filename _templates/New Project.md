---
tags: []
status: planning
created: <% tp.date.now("YYYY-MM-DD") %>
---

```dataviewjs
const projName = dv.current().file.name;
const bracketLink = /\[\[([^\]|]+)(?:\|[^\]]+)?\]\]/g;

function linksToThisProject(text, sourcePath) {
  for (const m of String(text).matchAll(bracketLink)) {
    const file = dv.app.metadataCache.getFirstLinkpathDest(m[1].trim(), sourcePath);
    if (file && file.basename === projName) return true;
  }
  return false;
}

const pages = dv.pages('"Work log"').sort(p => p.file.name, "desc");
const groups = [];
for (const page of pages) {
  const tasks = page.file.tasks.where(
    t => !t.completed && t.status !== "-" && linksToThisProject(t.text, page.file.path)
  );
  if (tasks.length) groups.push({ page, tasks });
}

if (groups.length) {
  dv.header(2, "Tasks");
  for (const { page, tasks } of groups) {
    dv.header(4, "[[Work log/" + page.file.name + "|" + page.file.name + "]]");
    dv.taskList(tasks, false);
  }
}
```

## Overview
<!-- One paragraph: what this project is and why it matters. -->

## Links
-

## Goals
-

## Decisions
<!-- Key decisions and rationale. -->

## Open Questions
-

## Notes
