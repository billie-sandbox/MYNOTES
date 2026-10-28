---
created: 2026-10-26
aliases:
  - Guide
  - How this vault works
---
# Vault Guide

How this vault is organized and how to use it. The structure is adapted from [kepano's vault](https://stephango.com/vault), plus a pipeline for video content.

> [!abstract] The one idea
> **Notes are organized by properties and links, not folders.** Every note says what it is in its `categories` property (e.g. `[[Books]]`), and each category hub automatically lists every note that links to it. You never file things by hand.

## Quick start

Create a new note, then run **Templates: Insert template** (`Ctrl+P`) and pick the template. The note appears in its category hub immediately.

| I want to… | Template / command | Shows up in |
| --- | --- | --- |
| Log my day | **Daily notes: Open today's daily note** | `Daily/` |
| Make a note named with today's date (e.g. `2026-10-28 Team meeting`) | **Templater: Create Templates/New Dated Note**, then pick a template and type the name | its category |
| Note a project idea | Add a line in [[Project Ideas]]. Don't make a new note | [[Home]] |
| Jot a quick thought | **Unique note creator: Create new unique note** | [[Journal]] |
| Capture an idea | Evergreen Template | [[Evergreen]] |
| Plan a long video | Long-form Video Template | [[Content]] |
| Plan a short / reel | Short-form Video Template | [[Content]] |
| Save an article | Clipping Template | [[Clippings]] |
| Add a person | People Template | [[People]] |
| Write meeting notes | Meeting Template | [[Meetings]] |
| Track a project | Project Template | [[Projects]] |
| Log a book / movie / show / album / game | Book, Movie, Show, Album, Video Game… Template | [[Books]], [[Movies]], [[Shows]], [[Albums]], [[Games]] |
| Save a place / trip | Place, City, Restaurant, Trip Template | [[Places]], [[Trips]] |
| Save a recipe / product | Recipe, Product Template | [[Recipes]], [[Products]] |

All templates live in `Templates/`. Ask Claude to create notes for you; it follows these same rules.

## Where things go

| Folder | What belongs there |
| --- | --- |
| `Notes/` | My own writing: ideas, meetings, trips, projects, videos |
| `References/` | Things in the world: books, people, places, movies, companies |
| `Clippings/` | Articles written by others, saved from the web |
| `Daily/` | Daily notes, named `YYYY-MM-DD` (created automatically) |
| `Categories/` | One hub note per category — don't put anything else here |
| `Templates/` | Note templates; `Templates/Bases/` holds the views behind every hub |
| `Attachments/` | Images and PDFs (pasted files land here automatically) |
| *root* | Journal entries (`YYYY-MM-DD HHmm`) and this guide |

**Daily notes:** each one starts with today's date in `created`, a **Todo** checklist (type `- [ ] `), and a **Notes** section to write in. At the bottom, **Today's notes** lists everything else made that day. Unfinished todos don't roll over by themselves, so copy them to the next day.

**Home and project ideas:** [[Home]] (bookmarked) is the dashboard: today's daily note, active projects, project ideas, content in progress and recent edits. Every raw project idea goes into the single [[Project Ideas]] note as one dated line. When an idea is worth planning, give it its own note from the Project Template (`status: idea` → `active` → `paused`/`done`) and link it from Project Ideas.

If unsure: my own thoughts go in `Notes/`, things that exist in the world go in `References/`.

## The rules

1. **What a note *is* → a link in `categories`.** `categories: ["[[Books]]"]`, not a `#books` tag.
2. **Related things → links too.** Author, genre, people, location, topics, platform: all `[[links]]`. This is what connects notes and fills the hub views.
3. **Links to notes that don't exist yet are fine.** They're breadcrumbs; create the note later if it becomes useful.
4. **Tags only for a state or format:** `#to-read`, `#0🌲` (evergreen idea), `#journal`, `#daily` / `#monthly`, `#inbox` (capture notes like [[Project Ideas]]).
5. **Category names are plural:** Books, People, Places.
6. **Dates are `YYYY-MM-DD`,** so daily notes can find everything from that day.
7. **Ratings go from 1 to 7.**
8. **Note names are the title,** written normally with spaces. Evergreen ideas are titled as a statement, e.g. *Writing is telepathy*.

> [!warning] Don't
> - **Don't create subfolders.** Use a category or a `type` link instead.
> - **Don't move or rename notes outside Obsidian** (File Explorer, other apps). Inside Obsidian, links update automatically; outside, they break.
> - **Don't edit files in `Templates/Bases/`** unless you mean to change that view everywhere it's used.
> - **Don't type over the stage values** in content notes; pick the exact text below.

## Content pipeline

Every video is one note in `Notes/` with its script inside. Open [[Content]] to see the board.

| Stage | Meaning |
| --- | --- |
| `1 Idea` | Just the concept |
| `2 Scripting` | Writing hook, outline, script |
| `3 Filming` | Recording |
| `4 Editing` | Cutting and polishing |
| `5 Scheduled` | Uploaded, waiting to go live |
| `6 Published` | Live; add the `url` |

To move a video forward, change its `stage` property. The numbers keep the board in order, so type the value exactly.

- `type` — `[[Long-form video]]` or `[[Short-form video]]` (the template fills this in).
- `platform` — `[[YouTube]]`, `[[TikTok]]`, `[[Instagram]]`…
- `publish` — the planned (later: actual) publish date; the board shows days left.
- `from` (shorts only) — link the long video a short was cut from.
- `topics` — link topics; putting `![[Content.base#Topic]]` in a topic note lists all videos about it.

## Hubs

Content: [[Content]]
Ideas & writing: [[Evergreen]] · [[Journal]] · [[Clippings]]
People & work: [[People]] · [[Companies]] · [[Meetings]] · [[Projects]] · [[Events]]
Media: [[Books]] · [[Movies]] · [[Shows]] · [[Albums]] · [[Podcasts]] · [[Podcast episodes]] · [[Games]] · [[Board games]]
Places & life: [[Places]] · [[Trips]] · [[Recipes]] · [[Products]]

## Sync

The vault backs up to GitHub automatically through the **Git** plugin:

- 5 minutes after I stop editing, it commits and pushes.
- It pulls every 10 minutes and when Obsidian opens.
- **Before closing Obsidian**, run **Git: Commit-and-sync** (`Ctrl+P`) to save the last few minutes of edits.
- On a new computer: clone `https://github.com/billie-sandbox/MYNOTES`, open it as a vault, and enable the Git plugin.
