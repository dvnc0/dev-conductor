---
name: code-review-and-quality
description: Multi-axis code review of a diff, pull request, or set of files in any language. Covers correctness, tests, readability, architecture, security, and performance, and returns findings ranked blocker / major / minor with path:line and a concrete failure scenario. Use before merging a change, when reviewing code written by you, another agent, or a person, or when asked to review a diff or PR, even one that looks small.
---

# Code Review and Quality

A review answers two questions: **does this change do what it claims, safely?** and **does it leave the codebase healthier than it found it?** It does not ask "is this how I would have written it?" Approve a change that clearly improves things even if it isn't perfect. Block a change that is wrong, unsafe, or makes the codebase meaningfully harder to work in.

This skill is language-agnostic. The process is the same in every stack. Stack-specific smells, tooling, and grep seeds live in the references:

| Reference | Load when |
|---|---|
| [review-axes.md](references/review-axes.md) | Always. The detailed questions and cross-language smells for each axis. |
| [performance.md](references/performance.md) | The change touches data access, loops over collections, I/O, concurrency, rendering, or anything on a hot path. |
| [language-notes.md](references/language-notes.md) | To find the project's test, lint, and type-check commands, and the footguns specific to the languages in the diff. |

For security, run the security axis here at the depth a general reviewer can. If the change touches untrusted input, auth, secrets, storage, or third-party integrations, say it needs a dedicated pass with `security-and-hardening` (or the `security-review` agent) instead of trying to do both jobs in one review.

## Step 0: Establish the inputs

Before reading code, pin down:

1. **Intent.** What is this change supposed to do? Check the task, spec, PR description, linked issue, or commit messages. If there's no stated intent, infer it from the diff and say that you inferred it.
2. **Scope.** The exact diff under review: `git diff <base>...HEAD`, `git diff --staged`, a PR, or a named list of files. Get `--stat` first to see the shape. Note generated, vendored, lockfile, and snapshot files; skim them rather than line-reviewing them.
3. **Stack.** Identify languages and frameworks from file extensions and manifests (`package.json`, `pyproject.toml`, `go.mod`, `pom.xml`, `build.gradle*`, `*.csproj`, `Gemfile`, `Cargo.toml`, `composer.json`, etc.). Load the matching sections of `language-notes.md`.
4. **Conventions.** The project's formatter and linter config, CONTRIBUTING or style docs, and above all the neighbouring code. The codebase's existing patterns are the standard, not your preferences.
5. **Verification story.** How is this checked? Find the test, lint, and type-check commands (CI config, manifest scripts, Makefile, task runner). Note what the author says they ran.

## Process

1. **Map the change.** For each changed symbol (function, type, endpoint, config key, schema), find its callers and consumers with grep. A change is only as safe as its least-prepared caller. Note any public surface: exported APIs, CLI flags, wire formats, DB schemas, events.
2. **Read the tests first.** They tell you what the author thinks the code does. Do they exist, test behavior rather than implementation, cover the edges and error paths, and fail if the change were reverted?
3. **Walk the axes** in `review-axes.md` over every changed file: correctness, tests, readability and simplicity, architecture and fit, security, performance and resources. Read enough surrounding code to judge each change in context, not just the changed lines.
4. **Verify cheaply when allowed.** Run the existing type checker, linter, and the relevant tests if your role permits running commands that don't modify the repo. Report real output. Never claim something passes that you didn't run, and say so when you couldn't run it.
5. **Try to refute your own findings.** Before writing one up, look for the guard upstream, the framework default, the caller that never passes that input, the test that already covers it. Drop what doesn't survive.
6. **Write the review** in the output format below, most severe first.

## Evidence bar

Every finding needs:
- **Location:** `path:line` (or a range), in the code under review.
- **The problem:** one or two sentences, specific.
- **Why it matters:** for correctness and security, a concrete failure scenario (this input or state, this sequence, leads to this wrong outcome). For maintainability, the concrete cost (who will misread it, what change will now be harder, what duplicate will drift).
- **A fix direction:** the smallest change that resolves it. A direction, not a rewrite.
- **Confidence** when it isn't obvious: verified (traced or ran it) vs. inferred.

Not findings:
- Style the formatter or linter already enforces.
- Preferences with no concrete cost ("I'd use a map here").
- Problems in unchanged code, unless the change depends on them or makes them worse. Report those as **pre-existing**, separately, and don't block on them.
- Speculation you couldn't tie to a real path. Either do the work to substantiate it or put it under Open questions.

## Severity

Use the same scale as the conductor and review agents:

| Severity | Meaning | Examples |
|---|---|---|
| **blocker** | Wrong or unsafe. Must be fixed before merge. | Wrong result on a normal input; data loss or corruption; crash on a reachable path; security hole; breaking a public contract with no migration; tests that pass while the feature is broken. |
| **major** | Should be fixed before merge. Real cost, but not immediately wrong. | Unhandled error path that will bite in production; missing tests for new behavior; a race under realistic timing; N+1 or unbounded query on a growing table; an abstraction or duplicate that will clearly drift; a change that fights the existing architecture. |
| **minor** | Optional. Author may decline. | Naming, small readability wins, a missing edge-case test for an unlikely input, a comment that would help. |

Severity is impact if the scenario happens. Mention likelihood separately when it changes urgency. Don't inflate a minor to be heard, and don't soften a real blocker to be polite.

## Change shape

Flag these as findings in their own right when they apply:
- **Too big to review well.** Roughly more than 400 changed lines of hand-written code, or many unrelated files. Say your review depth was reduced and suggest a split (by layer, by feature slice, or contract-first then consumers).
- **Mixed intent.** A refactor and a behavior change in one diff, so you can't tell which lines change behavior. Suggest separating them.
- **Unexplained public changes.** Renamed or removed exports, changed wire or storage formats, new required config, with no note on migration or compatibility.
- **New dependency.** Is it needed (does the stdlib or an existing dependency already do this)? Is it maintained, reasonably sized, and license-compatible? Is the version pinned the way the project pins others?
- **Dead code left behind.** Code the change made unreachable or unused. List it and ask whether to remove it; don't assume.

## Output format

```markdown
**Verdict:** approve | approve with changes | request changes

**Findings**
1. [blocker] `path/to/file.ext:42` — <problem>.
   Scenario: <input/state → wrong outcome>.
   Fix: <smallest fix>.
2. [major] ...
3. [minor] ...

**Pre-existing (not blocking):** <issues in code this change touches but didn't cause>

**Checked:** <axes covered, files read, commands run and their real results>
**Not checked:** <what you skipped and why: generated files, out-of-scope modules, couldn't run tests>
**Open questions:** <things only the author or user can answer>
```

If the change is good, say so in one line, list what you checked, and stop. A short honest approval is a valid review.

## Shortcuts that fail

| Tempting shortcut | Why it fails |
|---|---|
| Reviewing only the changed lines | Most bugs are in the interaction between new code and its callers, config, or data. |
| "Tests pass, so it works" | Tests only cover what the author thought of. Check what they *don't* exercise. |
| Trusting the PR description | The description is the claim. The diff is the evidence. Review the gap. |
| Flagging everything you'd do differently | Noise buries the one finding that matters and trains authors to ignore reviews. |
| Calling it a blocker to get attention | Severity inflation makes the scale worthless. Rate honestly. |
| "Probably fine" on code you didn't read | Either read it or list it under Not checked. |
| Approving AI-written code on fluency | Generated code reads confidently whether or not it's right. Hold it to the same evidence bar. |
| Proposing a rewrite as the fix | Reviews propose the smallest fix. Larger redesigns go in Open questions for the user to decide. |

## Before you return

- [ ] Intent is stated (given or inferred, and you said which)
- [ ] Callers of changed public symbols were checked
- [ ] Tests were read, and gaps noted
- [ ] Every axis in `review-axes.md` was considered for every changed file
- [ ] Every finding has `path:line`, a scenario or concrete cost, and a fix direction
- [ ] Every finding survived your own attempt to refute it
- [ ] Commands you report were actually run, with real output
- [ ] Pre-existing issues are separated from issues this change introduced
- [ ] "Checked" and "Not checked" are both filled in
