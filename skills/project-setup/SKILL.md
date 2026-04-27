---
name: project-setup
description: Sets up Claude Code project configuration from scratch. Use this skill whenever the user asks to initialize Claude project config, set up .claude directories, create settings.json, or scaffold Claude project structure. Trigger even if the user phrases it casually, e.g. "set up claude for this project" or "initialize claude config here".
---

# Project Setup

Creates the standard Claude Code project configuration structure.

## Steps

1. Create `.claude/` at project root if missing
2. Create `.claude/settings.json` if missing — leave empty (`{}`) unless the user specifies values
3. Create `.claude/memories/` if missing
4. Create `.claude/tasks/` if missing

## Rules

- Only update `.claude/settings.json` if the user explicitly requests specific values
- Do not create any other files or directories unless asked
- Report what was created vs what already existed
