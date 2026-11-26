# CLAUDE.md

Instructions for Claude Code in this repository. The rules shared by every AI agent are in `AGENTS.md` (OpenCode reads it directly); they are imported here so Claude Code follows the same rules. The glossary is imported because every task uses its words.

@AGENTS.md

@docs/glossary.md

## Claude Code specifics

- Read on demand, by role. Before any change: `docs/requirements.md` (the parts the task touches), `docs/architecture/README.md` and `docs/architecture/decisions.md`. For `apps/api`: also `docs/architecture/backend.md` and `docs/architecture/data-model.md`. For `apps/web`: also `docs/architecture/frontend.md`.
- Subagents live in `.claude/agents/`. `architect` is read-only: give it a plan, or the list of changed files, and it checks them against `docs/architecture/`. Use it before implementing a story and before reporting the story as done.
- OpenCode keeps its own configuration (`opencode.jsonc`, `.opencode/`). When a doc path or a shared rule changes, update both tools' files in the same change.
