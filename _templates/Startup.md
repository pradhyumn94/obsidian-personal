<%*
// Auto-create this week's work log if missing (Mon–Wed catch-up if you skipped Monday).
try {
    const dow = moment().day(); // 0=Sun … 6=Sat
    const shouldEnsureWeekFile = dow >= 1 && dow <= 3; // Mon–Wed only

    if (shouldEnsureWeekFile) {
        const weekStart = tp.date.weekday("YYYY-MM-DD", 1);
        const filePath = `Work log/${weekStart}.md`;
        const exists = app.vault.getAbstractFileByPath(filePath);

        if (!exists) {
            let template = tp.file.find_tfile("_templates/Weekly Work Log.md");
            if (!template) template = tp.file.find_tfile("Weekly Work Log.md");
            const folder = app.vault.getAbstractFileByPath("Work log");
            if (!template || !folder) {
                console.error("Startup template error: missing Weekly Work Log template or Work log folder");
            } else {
                // open_new_note = false: don't try to open a leaf during startup
                await tp.file.create_new(template, weekStart, false, folder);
            }
        }
    }
} catch (e) {
    console.error("Startup template error:", e);
}
_%>
