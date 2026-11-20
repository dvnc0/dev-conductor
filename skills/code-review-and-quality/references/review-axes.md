# Review Axes

Detailed questions for each axis of the `code-review-and-quality` skill. These are prompts for reading, not a checklist to tick: an axis that turns up nothing after an honest look is fine. Grep seeds are starting points across common stacks; adapt them to the code in front of you and always read the surrounding code before claiming anything.

---

## 1. Correctness

Does the code do what the change claims, for every input it can actually receive?

**Trace one real example by hand.** Pick a realistic input and walk it through the changed code to the output. Then pick an edge input and do it again.

### Logic
- Conditions inverted, `<` vs `<=`, `&&` vs `||`, negation applied to the wrong part of an expression.
- Off-by-one on ranges, slices, pagination, and loop bounds. Inclusive vs exclusive ends.
- Early returns or `continue`s that skip a required step (cleanup, audit, cache invalidation).
- Switch/match without a default, or a default that silently does the wrong thing for a new enum value.
- Copy-paste blocks where one variable wasn't updated.

### Data and types
- **Absent vs empty vs zero vs default.** Are `null`/`None`/`nil`/`undefined`, empty string, `0`, `false`, and a missing key distinguished where it matters, or silently merged? Truthiness checks (`if (x)`, `if x:`) that treat `0` or `""` as missing are a classic.
- **Numbers.** Integer overflow, float used for money, precision lost in JSON (64-bit IDs in JavaScript), division by zero, `NaN` propagation, rounding mode.
- **Strings.** Unicode normalization, case-folding that's locale-dependent, byte length vs character length, truncation that splits a multi-byte character.
- **Time.** Timezone-naive values, DST, local vs UTC comparisons, clock skew between machines, durations stored in ambiguous units.
- **Collections.** Mutating a collection while iterating it; shared mutable defaults (Python default args, shared slices or maps in Go, static mutable fields); aliasing where a copy was intended.

### State and lifecycle
- What must be true before this runs, and who guarantees it?
- Can it run twice (retry, double-click, redelivered message)? Is it idempotent, or does the second run corrupt the first?
- Partial success: three of five writes land and then it throws. What's left behind, and can a retry recover?
- Cleanup on every exit path, including exceptions and early returns (`finally`, `defer`, `using`, `with`, RAII, `try-with-resources`).
- Caches: what invalidates them, and does this change write data a cache is serving stale?

### Concurrency
- Check-then-act and read-modify-write without a lock, transaction, or atomic operation.
- Shared state touched from multiple threads, goroutines, async tasks, or workers.
- Locks held across I/O, `await`, or callbacks; lock ordering that can deadlock.
- Ordering assumptions between events, messages, or async operations that the system doesn't actually guarantee.
- **Unawaited async work.** A promise, task, or future created and never awaited loses its errors and its ordering.

### Error handling
- Every external call (network, disk, DB, subprocess, parse) has a failure path. What happens on timeout, on error, on a success response with garbage in it?
- Errors swallowed, logged and ignored, or converted to a default that looks like success. **A failure that produces a wrong answer instead of an error is the most dangerous kind.**
- Errors that leak internals to the caller (stack traces, SQL, file paths).
- Retries without backoff, without a cap, or around a non-idempotent operation.

Cross-language grep seeds for swallowed or ignored errors:

| Stack | Seeds |
|---|---|
| JS/TS | `catch\s*\([^)]*\)\s*\{\s*\}`, `\.catch\(\s*\(\)\s*=>\s*(\{\s*\}|null|undefined)\)`, promises not awaited in `async` functions, `// @ts-ignore`, `as any` |
| Python | `except\s*:`, `except Exception\s*:\s*pass`, `except.*:\s*(pass|continue|return None)` |
| Go | `_ = `, `, _ :=` on calls returning `error`, `if err != nil \{\s*(return nil|continue)` |
| Java/Kotlin | `catch \((Exception|Throwable) \w+\) \{\s*\}`, `printStackTrace\(\)`, `runCatching` with ignored result |
| C#/.NET | `catch\s*(\(Exception\))?\s*\{\s*\}`, `async void`, `.Result`, `.Wait\(\)` |
| Ruby | `rescue\s*(=>\s*\w+)?\s*$`, `rescue nil`, `rescue Exception` |
| Rust | `.unwrap()`, `.expect(` outside tests and provably-safe spots, `let _ =` on a `Result`, `.ok();` discarding errors |
| PHP | `@` error suppression, `catch \(\\?Throwable \$\w+\) \{\s*\}` |
| Shell | missing `set -euo pipefail`, unquoted `$var`, `|| true` hiding failures |

---

## 2. Tests

Do the tests prove the change works and will catch it breaking later?

- **Exist for new behavior.** New branches, new error paths, and the bug being fixed (a regression test that fails without the fix).
- **Test behavior, not implementation.** Assert on outputs, state, and observable effects. Tests that assert private calls or mirror the implementation line by line break on every refactor and catch nothing.
- **Would fail if the change were reverted or broken.** Mentally delete the key line. Does a test go red? A test that can't fail is decoration.
- **Mocks in the right place.** Mocking the exact thing that would fail in production (the DB query, the HTTP response shape) hides the bug. Prefer fakes at real boundaries.
- **Edges and errors.** Empty, single, boundary, too large, malformed, unauthorized, dependency down.
- **Deterministic.** No reliance on wall-clock time, random seeds, test order, network, or shared global state without control. Sleeps used to "wait" for async work are flaky by design.
- **Names describe the behavior** under test, so a failure tells you what broke.
- **Snapshot tests** updated wholesale in the diff deserve a look: was the new snapshot reviewed, or just accepted?

---

## 3. Readability and simplicity

Can the next engineer or agent understand and safely change this without the author?

- **Names** say what something is or does, in the domain's language, consistent with the rest of the codebase. Vague names (`data`, `info`, `handle`, `process`, `util`, `temp`, `result2`) hide intent.
- **Control flow** is straightforward: guard clauses over deep nesting, no nested ternaries, no boolean parameters that flip behavior (`doThing(true, false)`), no flags threaded through five layers.
- **Functions** do one thing at one level of abstraction. A function that both orchestrates and does low-level work is two functions.
- **Cleverness** that saves three lines and costs a reader five minutes is a net loss.
- **Size.** Could the same behavior be expressed in substantially less code using what the language, framework, or codebase already provides?
- **Abstractions earn their place.** An interface with one implementation, a factory for one type, a config option nobody sets, a generic helper with one caller: these cost reading time now for flexibility that may never be used.
- **Comments** explain *why* (constraints, surprising decisions, links to issues), not *what* the code already says. Stale comments that contradict the code are worse than none.
- **Dead code:** commented-out blocks, unused parameters and variables, unreachable branches, `// removed` markers, compatibility shims with no remaining callers.

Structural fixes worth proposing by name when they apply:
- Replace a growing chain of conditionals on the same value with a lookup table, a dispatcher, or polymorphism.
- Collapse near-duplicate branches into one flow with the difference as data.
- Split orchestration (the sequence of steps) from the steps themselves.
- Delete an abstraction instead of polishing it, when it no longer pulls its weight.

---

## 4. Architecture and fit

Does the change fit the system it's going into?

- **Follows existing patterns** for the same kind of problem (error handling, data access, validation, logging, config, dependency injection). If it introduces a new pattern, is that justified, and will the old one now coexist forever?
- **Right layer.** Feature-specific logic leaking into shared or core modules; business rules in controllers, views, or SQL; I/O inside what should be pure logic.
- **Reuse.** Grep for an existing helper, component, or service that already does this. A near-duplicate of an existing utility will drift from it.
- **Dependency direction.** Lower layers importing higher ones, circular imports, a shared package importing an app package.
- **Boundaries and contracts.** Are public interfaces, wire formats, DB schemas, events, and config keys changed compatibly? Is there a migration path for existing data and callers? These are one-way doors: hard to undo once shipped.
- **Coupling.** How many places must change together to make the next change like this one? If the answer grew, the design got worse.
- **Configuration.** New settings have sane defaults, are documented where others are, and fail loudly if required and missing.
- **Observability.** Will an operator be able to tell this is failing? Errors logged with enough context (but no secrets), metrics or traces where the codebase already uses them.

---

## 5. Security (general-reviewer depth)

A general review checks the obvious; a dedicated security pass goes deep. Flag the change for `security-and-hardening` if it touches any of: authentication, authorization, sessions, secrets, crypto, file uploads, deserialization, shell or code execution, outbound requests to user-supplied URLs, multi-tenant data, or LLM/agent tool use.

Quick checks:
- Untrusted input (request data, files, env, third-party responses, model output) validated at the boundary before use.
- Queries parameterized; shell commands built from argument arrays, not strings; HTML output escaped by the framework, not bypassed.
- Authorization checked on the server for every new endpoint or action, including ownership of the object being touched.
- No secrets, tokens, or credentials in code, tests, fixtures, logs, or error messages.
- New dependencies from a trusted source, pinned like the others.

---

## 6. Performance and resources

Load [performance.md](performance.md) for detail. Quick checks:
- Queries or network calls inside loops (N+1).
- Unbounded reads: no `LIMIT`, no pagination, loading a whole file or table into memory.
- Blocking calls on an async event loop or UI thread.
- Collections, caches, or queues that only grow.
- Resources (connections, file handles, subscriptions, timers, goroutines, threads) opened without a guaranteed close.
- Missing timeouts on outbound calls.

Only flag performance issues with a plausible growth story or a hot path. "This could be faster" on code that runs once at startup is not a finding.
