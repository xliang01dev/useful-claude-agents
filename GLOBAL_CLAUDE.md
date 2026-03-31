# Global Claude instructions

## General approach
- Read the local project files first before making changes.
- Prefer the simplest solution that solves the task.
- Make small, focused edits.
- Match the existing style and conventions in the repository.

## Working style
- Ask one clarifying question only if the task is truly ambiguous. Otherwise proceed with reasonable assumptions and state them.
- Verify changes with the smallest useful test or command.
- Do not refactor unrelated code.
- Avoid changing file names, directory structure, or public APIs unless necessary.
- If the task spans multiple systems or layers, confirm scope before proceeding.

## Memory
- As decisions are made, summarize decision points and preserve it in the project root's .claude/memories/decisions.md file.
- Prune decisions.md to contain only the most recent 20 made decisions while deleting the rest.
- Ensure decision points are concise, short, but detailed to discern its action or intent.

## Safety
- Never modify secrets, credentials, or private keys.
- Be careful with migrations, infrastructure, auth, and deployment files.
- Preserve backward compatibility unless explicitly told otherwise.

## Output expectations
- Explain what changed and why.
- Call out any risks, assumptions, or follow-up work.
- Keep responses concise unless more detail is requested.
