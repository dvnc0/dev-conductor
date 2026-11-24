# /conductor

A Claude Code command that turns the main session into an orchestrator. The conductor doesn't do the work itself. It plans, delegates to role-specific subagents (research, UX, design, dev, QA, reviewers), runs teams of agents in parallel when the work splits cleanly, and gates every serious review finding through a fresh skeptic before it becomes work.

```
/conductor add CSV export to the reports page
```

## How it works

1. **Plan.** Breaks the task into steps with an agent, owner, and handoff for each. Non-trivial plans get a `skeptical-architect` pass to cut scope before you see them. Waits for your go-ahead.
2. **Research first.** `research` grounds the plan in the actual codebase. For user-facing work, `ui-ux` writes the flow and states, then `design` handles the visual treatment.
3. **Build.** `dev` implements and `qa` tests. Larger work can run as a team (`dev-1`, `dev-2`, …) with exclusive file ownership, a shared roster, and `SendMessage` for coordination. Optional git worktree isolation.
4. **Review gate.** `code-review` and `security-review` run in parallel, plus `ui-ux` for user-facing changes and `adversarial-review` for risky ones.
5. **Verify.** Every blocker or major finding goes to its own fresh `skeptical-reviewer`, which tries to prove it wrong. Only CONFIRMED findings go back to `dev`.
6. **Loop**, up to 3 rounds, then **report**: what changed, what was found, fixed, refuted, or deferred. Fixes go back to the same `dev`, and the reviewers that raised them re-check only the fix, both resumed with `SendMessage` rather than respawned.

## Models and cost

Opus goes where judgment matters most: reviewing, and verifying the findings that would block a merge. Sonnet does the bulk of the work.

| Agent | Default model / effort |
|---|---|
| `research`, `qa` | sonnet / medium |
| `ui-ux` | sonnet / medium |
| `design`, `dev` | sonnet / high |
| `code-review`, `security-review` | opus / medium |
| `adversarial-review` | opus / high (risky changes only) |
| `skeptical-architect` | opus / medium |
| `skeptical-reviewer` | sonnet / high, overridden to opus for blockers and security, concurrency, data, or auth majors |

The conductor can override model and effort on each spawn. It drops to `haiku` or lower effort for narrow lookups and mechanical edits. It uses an **opus `dev`** for cross-cutting changes, concurrency or subtle logic, security-sensitive fixes, code with no clear pattern to follow, or a fix that a Sonnet `dev` has already missed once. Every override is listed in the plan with its reason.

## Layout

```
commands/conductor.md                     the orchestrator (runs in the main session)

agents/research.md                        read-only discovery: files, patterns, constraints
agents/ui-ux.md                           flows, states, accessibility; spec before, review after
agents/design.md                          visual design; styles, tokens, prototypes only
agents/dev.md                             implementation
agents/qa.md                              tests only
agents/code-review.md                     multi-axis review       (preloads code-review-and-quality)
agents/security-review.md                 security review         (preloads security-and-hardening)
agents/adversarial-review.md              adversarial teardown    (preloads adversarial)
agents/skeptical-architect.md             scope and simplicity gut check
agents/skeptical-reviewer.md              tries to refute one finding

skills/code-review-and-quality/           review process + references/
    references/review-axes.md             per-axis questions, cross-language smells and grep seeds
    references/performance.md             N+1, unbounded reads, blocking, leaks, by stack
    references/language-notes.md          verification commands and footguns per language
skills/security-and-hardening/            build mode (secure defaults) + review mode (taint trails)
    references/vulnerability-catalog.md   OWASP Top 10 (2025) + API items: sources, sinks, grep, traps
    references/secure-defaults.md         the safe API per task across 8 stacks
    references/llm-and-agents.md          prompt injection, tool misuse, RAG, memory, agent trust
skills/adversarial/                       five-pass attack method + Kill Report
    references/attack-vectors.md          vectors by target type (code, idea, plan, writing, data)
```

The review skills are language-agnostic: process and evidence rules live in `SKILL.md`, and stack-specific material lives in the references, which are loaded on demand.

All reviewers report on the same scale: **blocker** (must fix), **major** (should fix before merge), **minor** (optional).

## Requirements

- **Claude Code** with subagent support. Tested with 2.1.329. Frontmatter fields such as `effort`, `permissionMode`, and `skills:` may need a recent version.
- **`frontend-design` skill (optional).** `ui-ux` and `design` load it for interface work. It ships in Anthropic's `example-skills` plugin and isn't included here. Without it those agents still work, just without that guidance.
- **A brand or design-system skill (optional).** If your project has one installed, `ui-ux` and `design` will use it.

If an agent is missing, the conductor falls back to `general-purpose` with the role described in the prompt, so a partial install degrades rather than breaks.

## Install

User scope (`~/.claude`), using symlinks so `git pull` updates everything. Run from the repo root, and check that none of the targets already exist first: these names are generic, and a symlink won't replace an existing file or directory.

```bash
SRC="$PWD"
mkdir -p ~/.claude/{commands,agents,skills}

for f in "$SRC"/commands/*.md "$SRC"/agents/*.md; do
  dest=~/.claude/$(basename "$(dirname "$f")")/$(basename "$f")
  [ -e "$dest" ] && echo "exists, skipping: $dest" || ln -sn "$f" "$dest"
done

for d in "$SRC"/skills/*/; do
  dest=~/.claude/skills/$(basename "$d")
  [ -e "$dest" ] && echo "exists, skipping: $dest" || ln -sn "${d%/}" "$dest"
done
```

The symlinks break if you move the repo. Re-run the script after moving it.

To install for a single project instead, copy (or symlink) the same folders into that project's `.claude/` directory. Project-level agents with the same `name:` override user-level ones.

## Usage

```
/conductor fix the race in the job queue retry logic
/conductor add dark mode to the settings page
/conductor refactor the auth middleware to support API keys; proceed without asking
```

The conductor asks for approval on the plan before delegating unless you tell it to proceed. It escalates scope changes, risky decisions, and findings it can't resolve.

The agents and skills also work on their own: ask for the `code-review` or `security-review` agent directly, or invoke a skill by name.

## Credits

- `skeptical-architect` is adapted from the "Zoom Out Goldfish" role in [google/elephant-goldfish-model](https://github.com/google/elephant-goldfish-model).
- The skill format (overview, process, shortcuts table, verification checklist) is inspired by [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills).
- `security-and-hardening` references are organized by the [OWASP](https://owasp.org) Top 10 (2025), API Security Top 10 (2023), Top 10 for LLM Applications, and Top 10 for Agentic Applications (CC BY-SA 4.0). This project is not affiliated with or endorsed by OWASP.
