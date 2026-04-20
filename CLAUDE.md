# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an **Obsidian vault** — a personal knowledge base of markdown notes, not a software project. There are no build steps, tests, or linting commands. Content is written in Chinese and English.

## Vault Structure

- **Daily/** — Daily notes, organized as `YYYY/M/YYYY-MM-DD.md` (template: `Templates/daily.template.md`)
- **Templates/** — Obsidian templates (`daily.template.md`, `blog.template.md`)
- **Canvas/** — Notes on 2D graphics editors, rendering engines, and canvas topics
- **BigFrontEnd/** — Frontend development notes (React, JavaScript, Electron, Next.js)
- **Rust/** — Rust programming notes
- **AI/** — AI-related notes and workflows
- **Excalidraw/** — Excalidraw diagram files (`.excalidraw.md`)
- **Clippings/** — Saved articles/web clippings

## Key Conventions

- Notes use Obsidian-flavored markdown: `[[wikilinks]]`, callouts, dataview queries, and LaTeX (via latex-suite plugin)
- Blog template frontmatter: `date`, `tags` fields
- Daily notes template: starts with a TODO checklist
- Vim mode is enabled (`.obsidian.vimrc`: `jj` mapped to Escape)

## Git

- Managed via the **obsidian-git** plugin for automatic vault backups
- `.obsidian/workspace.json` is gitignored (changes frequently)
- Commit messages follow the pattern: `vault backup: YYYY-MM-DD HH:MM:SS`

## When Editing Notes

- Preserve existing wikilinks (`[[...]]`) and Obsidian syntax
- Match the language (Chinese or English) of the surrounding content
- Do not modify `.obsidian/` config files unless explicitly asked
