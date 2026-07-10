<%*
if (tp.frontmatter.status === "done") {
  await tp.file.move("1 Projects/Archive/" + tp.file.title);
  new Notice("✓ Archived: " + tp.file.title);
} else {
  new Notice("Status is '" + tp.frontmatter.status + "' — set to done first.");
}
%>