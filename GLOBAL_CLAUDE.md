# Global Claude instructions

## General approach
- Read the local project files from PROJECT_ROOT first before making changes.
- Prefer the simplest solution that solves the task.
- Make small, focused edits.
- Match the existing style and conventions in the repository.

## Working style
- Ask one clarifying question only if the task is truly ambiguous. Otherwise proceed with reasonable assumptions and state them.
- Verify changes with the smallest useful test or command.
- Do not refactor unrelated code.
- Avoid changing file names, directory structure, or public APIs unless necessary.
- If the task spans multiple systems or layers, confirm scope before proceeding.

## Project Configuration
- Create a PROJECT_ROOT/.claude file if non-existent.
- Create settings.json config under PROJECT_ROOT/.claude if non-existent.
- Create these directories under PROJECT_ROOT/.claude if non-existent:
    - /memories - Memory files that persist across sessions
    - /tasks - Project tasks and actions

## Project Architecture and Design Decisions
- As project level architecture / design decisions are made, summarize them as concise and short bullet points in /memories/decisions.md file.
- When there are over 20 decision bullets in /memories/decisions.md, prune accordingly:
    - Summarize all decisions into 5 or less short and concise paragraphs, grouping similar decisions when possible
    - Store decisions summaries in /memories/decisions-summary.md
    - Clear /memories/decisions.md so new decision bullets can be stored
- Avoid storing decisions in conversations unrelated to architecture or product design

## Project Permissions
- Allow file creation and modification for PROJECT_ROOT/.claude/memories and PROJECT_ROOT/.claude/projects folders.
- When permissions are required for project level work, update PROJECT_ROOT/.claude/settings.json to grant or deny.

## Safety
- Never modify secrets, credentials, or private keys.
- Be careful with migrations, infrastructure, auth, and deployment files.
- Preserve backward compatibility unless explicitly told otherwise.

## Output expectations
- Explain what changed and why.
- Call out any risks, assumptions, or follow-up work.
- Keep responses concise unless more detail is requested.
