# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal Obsidian vault (Markdown notes), not a software project. There is no build, lint, or test step. Claude's role here is a note-taking assistant: drafting and editing notes, organizing structure, linking related notes, and maintaining vault config.

The structure is adapted from kepano's vault (https://github.com/kepano/kepano-obsidian, links-first version after commit d94cc1e), plus a content-creation pipeline for video. When new conventions are agreed with the user, record them here **and** update `Vault Guide.md` (the user-facing guide, bookmarked in Obsidian) so the two never disagree. This file is the source of truth for rules; the guide explains them for daily use.

## Vault structure

Organization comes from properties and links, not folders. Folders are flat (no subfolders except `Templates/Bases/`):

- `Notes/` — the user's own writing: ideas, meetings, trips, projects, content pieces.
- `References/` — things in the world: books, people, places, movies, companies.
- `Clippings/` — articles by others saved from the web.
- `Daily/` — daily notes named `YYYY-MM-DD` (created from `Templates/Daily Note Template`).
- `Categories/` — one hub note per category; mostly just embeds its Base (`![[Books.base]]`).
- `Templates/` — note templates; `Templates/Bases/` holds every `.base` file.
- `Attachments/` — images and PDFs.

Daily notes are for writing first: the template has `created: {{date}}`, a `## Todo` section (`- [ ]` checkboxes), `## Notes`, then `## Today's notes` embedding `![[Daily.base]]` (lists notes from that day). Bases can't query checkboxes, so unfinished todos are carried forward by hand.

Journal entries are created with Unique note creator (`YYYY-MM-DD HHmm`, Journal template) at the vault root.

### How a note is classified

- **What a note is → a link in `categories`**, e.g. `categories: ["[[Books]]"]`. The matching Base filters on `categories.contains(link("Books"))`, so a note appears in its category hub automatically. Never classify with tags.
- **Sub-types and related things → links too**: `type: ["[[Authors]]"]`, `author`, `genre`, `topics`, `loc`, `people`, `org`. Links to notes that don't exist yet are fine and intended.
- **Tags only for state/format**: `to-read`, `0🌲` (evergreen idea notes), `journal`, `daily`/`monthly`, `inbox` (capture notes), `categories` (on hub notes), and nested type tags like `movies/genres`.
- Category names are plural. Dates are `YYYY-MM-DD`. Ratings are 1–7.
- Property types are declared in `.obsidian/types.json`; add new properties there with the right type.

When creating a note, start from the matching template in `Templates/` so the properties are complete. Bases embedded with a `#View` that filters on `this` (e.g. `![[Meetings.base#Person]]` in a person note) show entries linked to the embedding note.

### Projects and the Home dashboard

- **Project ideas go in one note: `Notes/Project Ideas.md`** (tag `inbox`). Never create a new note per raw idea: add a dated bullet (`- YYYY-MM-DD idea`) at the top of its *Inbox* section. When an idea is worth planning, create a project note (Project Template, `categories: [[Projects]]`) and replace the bullet with a link under *Promoted*; dropped ideas move to *Parked*.
- Project `status` (multitext) values: `idea`, `active`, `paused`, `done`. Projects.base views: Table, Active, Ideas, Paused or done.
- `Home.md` (vault root, alias Dashboard) is the dashboard: embeds `Home.base` (views Today, Recently edited) plus `Projects.base#Active`, `Projects.base#Ideas`, `Content.base#In progress`. Built only from core Bases; no dashboard/homepage plugins. Home, Project Ideas and Vault Guide are bookmarked.

### Content pipeline

Long- and short-form video live in the `[[Content]]` category (`Categories/Content.md`, `Templates/Bases/Content.base`). Templates: `Long-form Video Template`, `Short-form Video Template` (script lives inside the content note).

- `stage` is plain text with numbered values so grouping sorts in pipeline order: `1 Idea`, `2 Scripting`, `3 Filming`, `4 Editing`, `5 Scheduled`, `6 Published`. Keep these exact strings.
- `type` is `[[Long-form video]]` or `[[Short-form video]]`; `platform` is links (`[[YouTube]]`, `[[TikTok]]`, `[[Instagram]]`); `publish` is the target/actual publish date; short-form `from` links the long video it was cut from.
- Content.base views: Pipeline (grouped by stage), In progress, Ideas, Long-form, Short-form, Published, plus `#Topic` and `#Platform` for embedding in topic/platform notes.

Left out of kepano's setup on purpose: map views (`Map.base`, needs the Maps plugin) and the `Posts` category (replaced by Content). His example notes were not copied.

## Syncing (git)

The vault syncs between machines via git, remote `origin` = https://github.com/billie-sandbox/MYNOTES.git (branch `main`). Only commit or push when the user asks.

- `.gitattributes` forces LF line endings for text and marks images/PDFs as binary — keep new files LF.
- `.gitignore` excludes per-machine Obsidian UI state (`.obsidian/workspace*.json`, cache), `.trash/`, and `.claude/settings.local.json`. Don't commit those.
- Empty folders are kept with `.gitkeep` files.

## Obsidian settings that affect edits

- New notes are created at the vault root by default; attachments go in `Attachments/` (embed with `![[filename.png]]`).
- `alwaysUpdateLinks` is on: Obsidian rewrites wikilinks when a note is renamed *inside Obsidian*. Renames/moves done from the shell do **not** update links — use the Obsidian CLI to move/rename, or grep for `[[Old Name` and fix references.
- Obsidian keeps its config in memory and may overwrite `.obsidian/*.json` edited on disk. After editing config files, reload the app: `obsidian eval code="app.commands.executeCommandById('app:reload')"`. `obsidian plugin:enable ... filter=core` does not persist for core plugins; use `app.internalPlugins.getPluginById('<id>').enable(true)` via `obsidian eval` instead.

Core plugins in use: Properties, Daily notes, Templates, Unique note creator, Bases, Canvas, Backlinks, Bookmarks, Sync.

Community plugins are **Templater** and **Git**. Don't rely on Dataview, Tasks, etc.

**Templater** (`templater-obsidian`, settings in `.obsidian/plugins/templater-obsidian/data.json`) is used for one thing only: `Templates/New Dated Note.md`, run via the command *Templater: Create Templates/New Dated Note*. It asks for a template and a name, renames the note to `YYYY-MM-DD <name>`, and fills in the chosen template (substituting `{{date}}`, `{{title}}`, `{{time}}`). All other templates stay core-Templates syntax (`{{date}}`); don't convert them to `<% %>`. `trigger_on_file_creation` is off so it never touches daily/unique notes.

**Git** (`obsidian-git`, settings in `.obsidian/plugins/obsidian-git/data.json`): while Obsidian is open the plugin auto commits and syncs 5 minutes after the last edit, pulls every 10 minutes, and pulls on startup. Its commits are titled `vault backup: <date>`. So:
- Expect commits to appear on their own; check `git status`/`git log` before making manual commits. A multi-file change made through the shell may be auto-committed before a manual commit happens.
- Plugin settings changes must be written to `data.json` and then picked up with `obsidian plugin:reload id=obsidian-git` (Obsidian overwrites `data.json` from memory otherwise).

## Obsidian CLI

`obsidian` is on PATH and talks to the running Obsidian app (it must be open). Useful: `obsidian vault`, `obsidian search query=...`, `obsidian create name=... template=... silent`, `obsidian open path=...`, `obsidian eval code=...`. Run `obsidian help` for the full list.

## Skills

The `obsidian@obsidian-skills` plugin (kepano/obsidian-skills) is enabled in `.claude/settings.json`. Use its skills for the matching work:
- `obsidian:obsidian-markdown` — any `.md` note editing (wikilinks, callouts, embeds, frontmatter properties, tags).
- `obsidian:obsidian-bases` — `.base` files (table/card views over notes).
- `obsidian:json-canvas` — `.canvas` files (mind maps, visual boards).
- `obsidian:obsidian-cli` — vault operations via the Obsidian CLI (search, create, move with link updates); requires Obsidian running.
- `obsidian:defuddle` — clipping a web page into clean Markdown for a note.
- `obsidian:knap` — generating notes in bulk from templates + JSON/CSV data.

## Writing conventions

- Use Obsidian wikilinks (`[[Note Name]]`, `[[Note Name#Heading]]`, `[[Note Name|alias]]`) for internal links, not Markdown relative links.
- Note filenames are the note titles (human-readable, spaces allowed); avoid characters Obsidian disallows in filenames: `* " \ / < > : | ? # ^ [ ]`.
- Put metadata in YAML frontmatter properties rather than inline fields, so Bases can query it.
