<%*
// Templater: names the note "YYYY-MM-DD <name>" and fills it from a template you pick.
// Run with "Templater: Create Templates/New Dated Note" (not with core Templates: Insert template).
const self = tp.config.template_file.path;
const templates = app.vault.getMarkdownFiles()
  .filter(f => f.path.startsWith("Templates/") && !f.path.startsWith("Templates/Bases/") && f.path !== self)
  .sort((a, b) => a.basename.localeCompare(b.basename));
const pick = await tp.system.suggester(templates.map(f => f.basename.replace(/ Template$/, "")), templates, false, "Which template?");
if (!pick) return;
const name = await tp.system.prompt("Note name (today's date is added in front)");
if (!name) return;
const date = tp.date.now("YYYY-MM-DD");
const title = `${date} ${name}`;
await tp.file.rename(title);
const body = await app.vault.read(pick);
tR += body.replaceAll("{{date}}", date).replaceAll("{{title}}", title).replaceAll("{{time}}", tp.date.now("HH:mm"));
-%>
