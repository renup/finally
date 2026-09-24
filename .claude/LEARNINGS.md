# Claude Code customization: when to use what

Notes from building this project's `.claude/` setup. Covers the custom-made building blocks Claude Code offers — agents/subagents, commands, skills, hooks, and plugins — with pros, cons, and concrete examples from this repo.

## Quick comparison

| Type | Triggered by | Runs in | Best for |
|---|---|---|---|
| **Subagent** | Explicitly invoked (by name or by the main agent's judgment) | A separate context window | Isolating a big, self-contained job so it doesn't fill up your main conversation |
| **Command** | User typing `/name` | The current conversation | A prompt you'd otherwise retype constantly |
| **Skill** | Automatically, when the description matches the task (or explicitly via `/name`) | The current conversation (usually) | Teaching Claude a procedure/reference it should reach for on its own |
| **Hook** | A lifecycle event (`Stop`, tool call, etc.) — not the model's choice | A shell command, outside the model's control | Guaranteeing something *always* happens on an event, independent of what the model decides to do |
| **Plugin** | N/A — it's a packaging/distribution mechanism | Wraps any of the above | Sharing a bundle of the above with a team via git, instead of copy-pasting files |

---

## Subagents (`.claude/agents/*.md`)

A subagent is a named persona with its own system prompt and (optionally) a restricted tool set, invoked via the `Agent` tool. It starts with **zero memory of the current conversation** — everything it needs must be in the prompt handed to it.

**Examples in this repo**: `reviewer` and `codex-reviewer` (`.claude/agents/`) — both exist purely to shell out to `codex exec` for an independent second-opinion review, rather than reviewing inline.

**Syntax** — `.claude/agents/reviewer.md`: YAML frontmatter (`name`, `description` — the description is what the main agent matches against to decide when to delegate to it) followed by the subagent's system prompt as plain Markdown body.

```markdown
---
name: reviewer
description: carry out comprehensive review of all changes since the last commit
---

This subagent carries out comprehensive review of all changes since the last commit using shell commands.
IMPORTANT: You should not review the changes yourself but rather you should run the following shell command to kick off codex - codex is a separate AI agent that will carry out independent review.
Run this shell command:
'codex exec "Please review all changes since the last commit and write feedback to planning/REVIEW.md"'
Do not run review yourself.
```

### Pros
- Keeps a large, noisy task (e.g., reading an entire diff and cross-referencing it against a 400-line PLAN.md) out of the main conversation's context.
- Can be given a narrower tool set than the main agent — e.g., a review subagent that can only read/grep, never edit, so it can't "fix" what it was asked to critique.
- Parallelizable — several subagents can run at once for independent sub-problems.
- Good for enforcing "don't do X yourself, always delegate to Y" (as `reviewer.md` does by forcing every review through `codex exec` for a genuinely independent opinion, rather than Claude grading its own work).

### Cons
- Cold start: no shared context, so the invoking prompt must restate everything relevant — terse prompts to a fresh subagent produce shallow, generic results.
- You lose visibility into its intermediate steps (by design) — you only see the final report, so debugging *why* it concluded something is harder.
- Overhead isn't free — spinning one up for a two-line lookup is slower and more expensive than just doing the lookup.
- Findings from a subagent are self-reported ("trust but verify") — it may describe what it intended to do rather than what it actually did.

### When to use
- The task is big enough that its raw tool output (long file reads, grep results, a huge diff) would bloat your main context and you won't need that raw output again — only the conclusion.
- You want a structurally independent check (a second reviewer that didn't write the code being reviewed).

### When *not* to use
- A one-off lookup or a small edit — just do it directly; the delegation overhead exceeds the task.
- A task that depends heavily on nuance already established in the conversation (a fresh subagent won't have it, and re-deriving it is wasteful).

---

## Commands (`.claude/commands/*.md`)

A command is a slash command (`/doc-review`) that expands to a fixed prompt template, optionally with `$ARGUMENTS` substituted in. It runs **in the current conversation** — no context isolation.

**Example in this repo**: `/doc-review <file>` (`.claude/commands/doc-review.md`) — expands to "review this planning doc and append a questions/clarifications/simplification section," with the filename slotted in.

**Syntax** — `.claude/commands/doc-review.md`: no frontmatter needed for a simple command, just plain Markdown; `$ARGUMENTS` is substituted with whatever the user types after `/doc-review`.

```markdown
Review the documentation file in planning folder named $ARGUMENTS and add any questions, clarifications or feedback to a new section at the end along with any opportunies to simplify
```

Invoked as: `/doc-review PLAN.md`

### Pros
- Zero cold-start cost — it runs with full access to the current conversation's context.
- Turns a multi-sentence instruction you'd retype often ("review this planning doc for gaps, add a section with questions...") into two words.
- Easy to read and edit — it's just a Markdown file, no code.
- Discoverable by teammates (`/doc-review` shows up in the command list) — self-documenting in a way an ad-hoc prompt in someone's head isn't.

### Cons
- Static — it's a prompt template, not logic; it can't branch, call APIs, or make decisions beyond what the prompt itself asks the model to do.
- Pollutes the current context if the resulting work is large (unlike a subagent, there's no isolation).
- Command sprawl — if every slightly-different request becomes its own command, the list gets long and overlapping (e.g., you don't need `/doc-review-planning` *and* `/doc-review-backend` — one command with an argument covers both, as this repo's does).

### When to use
- A repeated, well-defined request phrased as an instruction ("review X for Y", "summarize the diff since last release") where you want the *current* conversation's context, not an isolated one.
- Team-wide consistency — everyone runs the same review checklist instead of freehand prompts that drift.

### When *not* to use
- The task needs real control flow (loops, conditionals on tool output, multi-stage pipelines) — that's a job for a skill or a script, not a prompt template.
- The output would be huge and you don't want it cluttering the current conversation — use a subagent instead.

---

## Skills (`.claude/skills/*/SKILL.md`)

A skill is a packaged capability with a description that Claude matches against the task *automatically* — you don't have to remember to invoke it. It can also be triggered explicitly. Skills can carry reference material, scripts, and multi-step procedures, not just a single prompt.

**Example in this repo**: `cerebras` (`.claude/skills/cerebras/SKILL.md`) — "how to call an LLM via LiteLLM/OpenRouter with Cerebras as the inference provider." PLAN.md §9 explicitly directs the LLM integration to use this skill rather than hand-rolling the LiteLLM call each time.

**Syntax** — `.claude/skills/cerebras/SKILL.md`: frontmatter (`name`, `description` — the description is the trigger Claude matches against) followed by reference docs and code snippets in the body, not just a prompt:

```markdown
---
name: cerebras-inference
description: Use this to write code to call an LLM using LiteLLM and OpenRouter with the Cerebras inference provider
---

# Calling an LLM via Cerebras

## Setup
The OPENROUTER_API_KEY must be set in the .env file...
`uv add litellm pydantic`

## Code snippets

​```python
from litellm import completion
MODEL = "openrouter/openai/gpt-oss-120b"
EXTRA_BODY = {"provider": {"order": ["cerebras"]}}
response = completion(model=MODEL, messages=messages, reasoning_effort="low", extra_body=EXTRA_BODY)
​```
```

### Pros
- Self-triggering — the model reaches for it when the task matches, without the user needing to know it exists or type a command.
- Encodes non-obvious, project-specific "how" (exact provider config, exact model ID, gotchas) once, so it's not re-derived (and potentially gotten wrong) every time the topic comes up.
- Can bundle more than a prompt — reference docs, helper scripts, checklists — genuinely richer than a command.
- Reduces drift: without it, "call the LLM" could be implemented three different ways across a codebase by three different sessions.

### Cons
- Auto-triggering is a double-edged sword — a poorly-scoped description can fire when you didn't want it to (or fail to fire when you did), and debugging *why* a skill didn't trigger is less direct than "why didn't my command run" (commands are explicit).
- More upfront investment to write well — a good trigger description and clear procedure take real iteration, unlike a command's one-shot prompt.
- Overkill for something used once.

### When to use
- A procedure that should happen *every time* a certain kind of task comes up, whether or not the user remembers to ask for it by name (e.g., "any time you're about to call an LLM from this codebase, use this exact provider setup").
- Non-obvious domain/project knowledge worth centralizing (rate limits, exact model IDs, required structured-output shape) — exactly what the `cerebras` skill does for §9's OpenRouter/Cerebras requirement.

### When *not* to use
- A task that's naturally opt-in and rare — that's better as an explicit command so it's never accidentally auto-triggered.
- Something so simple it doesn't need "packaging" — a one-line reminder belongs in CLAUDE.md, not a skill.

---

## Hooks (`settings.json` → `hooks`, or bundled in a plugin)

A hook is a shell command wired to a Claude Code **lifecycle event** — a tool call, a session `Stop`, etc. — not something the model decides to do. It fires automatically whenever the event occurs, every time, regardless of what the model "wants."

**Example in this repo**: the `Stop` hook (documented in `.claude/README.md`, and formerly packaged standalone as the `independent-reviewer` plugin's `hooks/hooks.json` — since folded into this doc, see note at the end of this section) runs `codex exec "Review changes since last commit and write results to REVIEW.md..."` after *every* assistant turn ends — including `/clear`, `/resume`, and compaction, not just when a review is requested.

**Syntax** — either inline in `.claude/settings.json` under a `hooks` key, or as a standalone `hooks/hooks.json` inside a plugin (same shape either way): keyed by lifecycle event (`Stop`, `PreToolUse`, etc.), each with a list of hook groups, each containing one or more `command`-type hooks.

```json
{
    "hooks": {
        "Stop": [
            {
                "hooks": [
                    {
                        "type": "command",
                        "command": "codex exec \"Review changes since last commit and write results to REVIEW.md under planning folder\""
                    }
                ]
            }
        ]
    }
}
```

### Pros
- Guaranteed to run — unlike a skill (which the model chooses to invoke) or a command (which the user must remember to type), a hook fires unconditionally on its event. It's the only mechanism here that isn't dependent on the model deciding to act.
- Enforces policy the model can't talk itself out of — e.g., "every turn gets reviewed by an independent tool," with no risk of the model deciding a review isn't needed this time.
- Fully external to the model — it's a plain shell command, so it can do anything a script can do (call another CLI, hit an API, write files), not just generate text.

### Cons
- Runs unconditionally, which is exactly what makes it costly here: `codex exec` is a real API call, so *every single turn* — even `/clear` or a trivial exchange — burns real usage, whether or not a review was wanted.
- No judgment — it can't tell "this turn was a one-line typo fix" from "this turn rewrote the auth layer"; it fires the same way regardless.
- Harder to discover than a command or skill — it doesn't show up as something you can invoke by name; you have to know to look in `settings.json` or a plugin's `hooks/hooks.json` to realize it exists at all.
- Failure mode is silent-ish — if the underlying command (`codex` CLI here) isn't installed or misconfigured, the hook fails on every event, not just once.

### When to use
- Something must happen every time an event occurs, with no exceptions and no dependency on the model remembering to do it — e.g., auto-formatting on file write, or blocking a dangerous command pattern before it runs.
- The action doesn't need the model's judgment at all — it's mechanical (lint, format, log, notify).

### When *not* to use
- The action is genuinely conditional ("review this *if* it looks like a big change") — that needs judgment, which belongs in a command or skill the model reasons about, not an unconditional hook. (This is exactly this repo's open problem: the `Stop` hook reviews *every* turn indiscriminately, costing API usage even on trivial turns — a command like `/review` invoked only when wanted would avoid that.)
- The action is expensive (a paid API call, a slow script) and the event it's attached to is high-frequency — the cost multiplies with every occurrence, as it does here.

---

## Plugins (`.claude-plugin/`, e.g. `independent-reviewer/`)

A plugin isn't a new capability type — it's a **distribution unit** that bundles any combination of hooks, agents, commands, and skills into one installable package, discoverable via a **marketplace** (`.claude-plugin/marketplace.json`).

**Example in this repo**: `independent-reviewer/` at the repo root — a plugin containing just the `Stop` hook shown above (previously at `hooks/hooks.json`, now removed — see the note at the end of this section). It's registered in `.claude-plugin/marketplace.json` so any teammate can add the marketplace and install it, instead of hand-copying hook JSON into their own `settings.json`.

**Syntax** — a plugin is a directory with a `.claude-plugin/plugin.json` manifest (name/description/version), plus any of `hooks/hooks.json`, `agents/`, `commands/`, `skills/` alongside it. It's made discoverable by an entry in a marketplace's `.claude-plugin/marketplace.json`:

```json
// independent-reviewer/.claude-plugin/plugin.json
{
    "name": "independent-reviewer",
    "description": "Carry out an independent review of all changes since the last commit",
    "version": "1.0.0"
}
```

```json
// .claude-plugin/marketplace.json (repo root)
{
    "name": "renu-marketplace",
    "owner": { "name": "Renu", "email": "rpunjabi12@gmail.com" },
    "plugins": [
        {
            "name": "independent-reviewer",
            "source": "./independent-reviewer",
            "description": "Carry out an independent review of all changes since last commit",
            "version": "1.0.0",
            "author": { "name": "Renu" }
        }
    ]
}
```

### Pros
- Team-shareable by construction — checked into git once, installed by everyone via the marketplace, instead of each person recreating loose `agents/`, `commands/`, `skills/`, and hook config independently.
- Groups related pieces (a hook + the agent it calls + a command that surfaces it) so they travel together and can't drift out of sync with each other.
- Versioned (`version` field) — you can evolve the bundle and teammates can tell if they're behind.

### Cons
- Structural overhead for something that's really just one file — wrapping a single hook in a plugin + marketplace entry (as `independent-reviewer` does) is more ceremony than just documenting "add this to your `settings.json`."
- Another layer to reason about when debugging: is this hook coming from `settings.json` directly, or from an installed plugin? (This repo's `.claude/README.md` calls this out explicitly.)
- Marketplace + plugin + component (agent/command/skill/hook) is three levels of indirection for a reader trying to find where behavior actually lives.

### When to use
- More than one person needs the *exact same* set of customizations, and "copy these files into your `.claude/`" isn't good enough (people forget, files drift).
- The bundle is genuinely multi-part (a hook that depends on a specific agent, plus a command to invoke it manually) — packaging keeps them together.

### When *not* to use
- It's a single small thing used by one person on one machine — a loose file in `.claude/agents/` or `.claude/commands/` is simpler and just as effective; don't reach for a plugin+marketplace just because the mechanism exists.

---

## Rule of thumb

- **Explicit + big + should not pollute current context** → subagent.
- **Explicit + repeated + fits in current context** → command.
- **Implicit (should trigger itself) + procedural/reference knowledge** → skill.
- **Must happen every time, unconditionally, regardless of the model's judgment** → hook.
- **Needs to be shared as a unit across a team, versioned, and installed rather than copied** → plugin (wrapping any of the above).
- **None of the above** — if it's a one-off, or small enough to just say in plain language, it doesn't need any of these; a note in CLAUDE.md or just doing the task directly is enough. Not everything custom needs to become a subagent/command/skill/plugin.
