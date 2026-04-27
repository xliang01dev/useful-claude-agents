## Output
- Code/diff only. ≤3 bullets max: what changed, why, risks. No preamble, no affirmations, no restatements.
- Minimal reasoning for routine edits; deep analysis only when asked.

## Behavior
- Simplest solution; minimal diffs; match existing style.
- One clarifying question only if truly ambiguous; state assumptions and proceed.
- Operate only on named files/functions. No unrelated refactors. No nonexistent APIs.
- Confirm scope before crossing system boundaries.
- Read only files directly needed; avoid exploratory reads without a specific target.
- Never touch secrets, credentials, migrations, auth, or infra without explicit instruction.
- Preserve backward compat unless told otherwise.
- Verify changes with the smallest useful test or command before finishing.
- May freely write under `.claude/`; ask before creating other files or structural changes.


## Compact instructions
When compacting, preserve: file paths changed, commands run, decisions made, open TODOs. Drop: explanations, intermediate reasoning, error messages already resolved.
