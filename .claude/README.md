# .claude/ configuration

Claude Code customizations for this project.

## Agents (`agents/`)

Custom subagents, invoked via the `Agent` tool or by name.

- **`reviewer`** — Reviews all changes since the last commit. Delegates the actual review to `codex exec` (a separate AI CLI) rather than reviewing inline, writing findings to `planning/REVIEW.md`.
- **`codex-reviewer`** — Reviews `planning/PLAN.md` specifically for production readiness, also via `codex exec`, writing findings to `planning/REVIEW.md`.

> Note: `reviewer.md` invokes `codex exec: "..."` (with a colon), which is invalid CLI syntax — `codex exec` takes the prompt as a plain positional argument, no colon. `codex-reviewer.md` has the correct form. Worth fixing `reviewer.md` to match.

## Commands (`commands/`)

Custom slash commands.

- **`/doc-review <file>`** — Reviews a documentation file in `planning/` and appends a section with questions, clarifications, and simplification opportunities.

## Settings (`settings.json`)

- **Plugins**: `frontend-design`, `context7`, `playwright` (all from `claude-plugins-official`).
- **Hooks**: A `Stop` hook runs `codex exec` to review changes and write to `planning/REVIEW.md` after every assistant turn ends (including `/clear`, `/resume`, and compaction — not just on request). This calls out to the `codex` CLI and costs real API usage each time.

## Skills (`skills/`)

Project-local skills available via the `Skill` tool.

## Notes: hooks, plugins, and the marketplace

- **Hooks** can be defined directly in `settings.json` (as above), or bundled inside a plugin. Either location works the same way at runtime.
- **Plugins** are the packaging unit for sharing Claude Code customizations — a plugin can bundle hooks, agents/subagents, commands, and skills together, rather than distributing them as loose files. See `independent-reviewer/` at the repo root for an example plugin (a `hooks/hooks.json` + `.claude-plugin/plugin.json`).
- **The marketplace** (`.claude-plugin/marketplace.json` at the repo root) is what makes a custom plugin discoverable — it lists plugins Claude can see/use, and lets the whole team install them by checking the marketplace file into git rather than copying config by hand.

For a fuller pros/cons/when-to-use breakdown of agents, subagents, commands, skills, and plugins — with tangible examples from this repo — see [`LEARNINGS.md`](LEARNINGS.md).
