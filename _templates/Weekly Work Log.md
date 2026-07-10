<%*
const mon = tp.date.weekday("YYYY-MM-DD", 1);
const tue = tp.date.weekday("YYYY-MM-DD", 2);
const wed = tp.date.weekday("YYYY-MM-DD", 3);
const thu = tp.date.weekday("YYYY-MM-DD", 4);
const fri = tp.date.weekday("YYYY-MM-DD", 5);
const fence = "```";
const projects = `## Projects\n\n${fence}dataviewjs\nconst projects = dv.pages('"1 Projects"').where(p => p.status === "active" && !p.file.path.startsWith("1 Projects/Archive"));\nif (projects.length) {\n  dv.list(projects.file.link);\n} else {\n  dv.paragraph("No active projects this week.");\n}\n${fence}\n\n---\n\n`;
tR += `${projects}## Week of ${mon}\n\n### ${mon} — Monday\n\n---\n\n### ${tue} — Tuesday\n\n---\n\n### ${wed} — Wednesday\n\n---\n\n### ${thu} — Thursday\n\n---\n\n### ${fri} — Friday\n\n---`;
%>
