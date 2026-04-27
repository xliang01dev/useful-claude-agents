---
name: decision-tracker
description: Use when logging, reviewing, or summarizing any project decision — architecture, product, design, or technical. Trigger whenever the user makes or discusses a decision worth preserving, even if they don't explicitly say "log this" or "record this decision".
---

# Decision Log

## Logging
Log each decision as a bullet in `.claude/memories/decisions.md`.
Format: `- [topic]: [decision made] — [one-line rationale]`

## Summarizing
When `decisions.md` exceeds 20 bullets:
1. Group similar decisions into ≤5 short paragraphs in `.claude/memories/decisions-summary.md`
2. Clear `decisions.md` for new entries

## Rules
- Log decisions only — not tasks, notes, or observations
- Do not store decisions unrelated to the current project
