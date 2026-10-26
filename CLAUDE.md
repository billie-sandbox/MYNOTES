# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal Obsidian vault (Markdown notes), not a software project. There is no build, lint, or test step. Claude's role here is a note-taking assistant: drafting and editing notes, organizing structure, linking related notes, and maintaining vault config.

The vault is new and nearly empty — a folder structure and note conventions have not been settled yet. When the user asks to organize or add notes, propose a structure rather than assuming one, and record agreed conventions in this file so they persist.

## Syncing (git)

The vault syncs between machines via git (see `Welcome.md`). The workflow is pull before editing, commit + push after. Only commit or push when the user asks.

- `.gitattributes` forces LF line endings for text and marks images/PDFs as binary — keep new files LF.
- `.gitignore` excludes per-machine Obsidian UI state (`.obsidian/workspace*.json`, cache), `.trash/`, and `.claude/settings.local.json`. Don't commit those.

## Obsidian settings that affect edits

From `.obsidian/app.json`:
- New notes are created at the vault **root** by default (`newFileLocation: root`).
- Attachments (images, PDFs) go in `attachments/` — embed them with `![[filename.png]]`.
- `alwaysUpdateLinks` is on: Obsidian rewrites wikilinks when a note is renamed *inside Obsidian*. Renames/moves done from the shell do **not** update links — when moving or renaming notes outside Obsidian, grep for `[[Old Name` and fix references, or use the Obsidian CLI.

Enabled core plugins worth knowing (`.obsidian/core-plugins.json`): Properties (YAML frontmatter), Daily notes, Templates, Bases (`.base` database views), Canvas (`.canvas`), Backlinks, Bookmarks, Sync.

The only community plugin is **Git** (`obsidian-git`, settings in `.obsidian/plugins/obsidian-git/data.json`). Don't rely on Dataview, Templater, etc. While Obsidian is open the plugin auto commits and syncs 5 minutes after the last edit, pulls every 10 minutes, and pulls on startup. Its commits are titled `vault backup: <date>`. So:
- Expect commits to appear on their own; check `git status`/`git log` before making manual commits.
- Plugin settings changes must be written to `data.json` and then picked up with `obsidian plugin:reload id=obsidian-git` (Obsidian overwrites `data.json` from memory otherwise).

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
