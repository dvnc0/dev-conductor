# Language Notes

Per-ecosystem notes for the `code-review-and-quality` skill: where to find the project's verification commands, and the footguns that show up most often in review. Load only the sections for the languages in the diff.

**Finding the commands.** Always prefer what the project already defines over the generic commands below. In order, check: CI config (`.github/workflows/*`, `.gitlab-ci.yml`, `Jenkinsfile`, `.circleci/`, `azure-pipelines.yml`), task runners (`Makefile`, `justfile`, `Taskfile.yml`, `package.json` scripts, `tox.ini`, `noxfile.py`), and CONTRIBUTING or README. Run only commands that don't modify the repo (no `--fix`, no formatter writes, no installs), and only if your role allows running commands.

---

## JavaScript / TypeScript

- **Commands:** `package.json` scripts (`test`, `lint`, `typecheck`, `check`). Common: `tsc --noEmit`, `eslint .`, `vitest run`, `jest`, `npx playwright test`. Lockfile tells you the package manager (`package-lock.json` npm, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`/`bun.lock`).
- **Footguns:**
  - Unawaited promises: errors vanish and ordering breaks. `forEach` with an `async` callback doesn't wait.
  - `==` coercion; truthiness treating `0` and `""` as missing; `||` defaults that clobber valid falsy values (use `??`).
  - `any`, `as` casts, non-null `!`, and `@ts-ignore` silencing the type checker exactly where it would have helped.
  - Numbers above 2^53 losing precision (IDs from other systems, money in minor units).
  - Mutating props, state, or shared objects in place; array methods that mutate (`sort`, `reverse`, `splice`).
  - `Date` parsing of non-ISO strings is implementation-dependent; month is zero-based.
  - React: missing or wrong effect dependencies, keys from array index on reorderable lists, state updates after unmount.

## Python

- **Commands:** `pyproject.toml` (`[tool.pytest]`, `[tool.ruff]`, `[tool.mypy]`), `tox.ini`, `noxfile.py`, `Makefile`. Common: `pytest`, `ruff check .`, `mypy .` or `pyright`, `python -m unittest`.
- **Footguns:**
  - Mutable default arguments (`def f(x=[])`) shared across calls.
  - Bare `except:` or `except Exception: pass` hiding errors, including `KeyboardInterrupt` for the bare form.
  - Late-binding closures in loops (`lambda: i` captures the last `i`).
  - Sync I/O inside `async def` blocking the event loop; forgetting `await` on a coroutine (it never runs).
  - Naive vs aware `datetime`s compared or mixed; `datetime.utcnow()` returns a naive value.
  - `is` used for value comparison; `float` for money (use `Decimal`).
  - Django/SQLAlchemy lazy loads in loops (see performance.md).

## Go

- **Commands:** `go build ./...`, `go vet ./...`, `go test ./...` (add `-race` for concurrency changes), `staticcheck ./...` or `golangci-lint run` if configured.
- **Footguns:**
  - Ignored `error` returns (`_ =`, `, _ :=`); errors wrapped without `%w` so `errors.Is` stops working.
  - Goroutine leaks: blocked sends or receives with no cancellation; missing `ctx.Done()` handling.
  - Data races on maps and shared structs (maps are not safe for concurrent writes).
  - Loop variable capture in goroutines and closures (fixed by per-iteration semantics in Go 1.22+, so check the `go` directive in `go.mod`).
  - `defer` inside a loop delays cleanup until the function returns.
  - Nil interface vs interface holding a nil pointer (`err != nil` true for a typed nil).
  - `http.DefaultClient` and `http.Get` have no timeout.
  - Slices sharing a backing array after `append`; returning a sub-slice of a large buffer keeps it alive.

## Java / Kotlin

- **Commands:** Maven `mvn -q verify` / `mvn test`; Gradle `./gradlew check` / `./gradlew test`. Linters if configured: Checkstyle, SpotBugs, PMD, Error Prone, detekt and ktlint (Kotlin).
- **Footguns:**
  - Swallowed exceptions; catching `Exception` or `Throwable` broadly; `printStackTrace()` instead of logging.
  - `equals`/`hashCode` inconsistent, or overridden one without the other; `==` on strings or boxed numbers.
  - Resources not in try-with-resources.
  - `Optional.get()` without a check; Kotlin `!!` on platform types from Java.
  - Mutable static state; non-thread-safe classes (`SimpleDateFormat`, `HashMap`) shared across threads.
  - JPA lazy loading outside a transaction (`LazyInitializationException`) or in loops (N+1).
  - Spring: `@Transactional` on private methods or self-invocation (proxy bypassed, so no transaction).

## C# / .NET

- **Commands:** `dotnet build`, `dotnet test`, `dotnet format --verify-no-changes` (read-only check). Analyzers run during build if configured (`Directory.Build.props`, `.editorconfig`).
- **Footguns:**
  - `async void` outside event handlers (exceptions crash the process); `.Result` / `.Wait()` on tasks (deadlocks, thread-pool starvation).
  - `IDisposable` not disposed (`using` missing), especially `HttpClient` created per request (socket exhaustion; use `IHttpClientFactory` or a shared instance).
  - Nullable reference warnings suppressed with `!`.
  - EF Core: queries materialized early (`ToList()` before `Where`), client-side evaluation, N+1 on navigation properties.
  - `DateTime` kinds mixed (`Local` vs `Utc` vs `Unspecified`); prefer `DateTimeOffset` for instants.

## Ruby / Rails

- **Commands:** `bundle exec rspec` or `bin/rails test`, `bundle exec rubocop`, `bundle exec brakeman` (security) if present.
- **Footguns:**
  - `rescue => e` swallowing everything; `rescue nil` modifiers.
  - N+1 via associations in views and serializers (`includes`, or the `bullet` gem if the project uses it).
  - Callbacks (`after_save`, etc.) with hidden side effects; `update_column` / `update_all` skipping validations and callbacks.
  - Mass assignment without strong parameters.
  - `Time.now` vs `Time.current` (zone-aware) in Rails.
  - Missing database constraints backing model validations (uniqueness validations race without a unique index).

## Rust

- **Commands:** `cargo build`, `cargo test`, `cargo clippy -- -D warnings`, `cargo fmt --check`.
- **Footguns:**
  - `unwrap()` / `expect()` on paths reachable by real input (panics in production).
  - `let _ = ...` silently discarding a `Result`.
  - Blocking calls inside async runtimes (use `spawn_blocking`); holding a `std::sync::Mutex` guard across `.await`.
  - `unsafe` blocks: each needs a comment justifying the invariants it relies on.
  - Integer overflow wraps silently in release builds; use `checked_*` / `saturating_*` where it matters.
  - Excessive `.clone()` hiding an ownership design problem (usually minor).

## PHP

- **Commands:** `composer test` or `vendor/bin/phpunit` / `vendor/bin/pest`, `vendor/bin/phpstan analyse`, `vendor/bin/psalm`, `vendor/bin/php-cs-fixer fix --dry-run`.
- **Footguns:**
  - Loose comparison `==` (`"abc" == 0` was true before PHP 8); use `===`.
  - `@` error suppression.
  - `unserialize` on anything user-influenced.
  - Laravel/Eloquent N+1 (`with()` eager loading); `$guarded = []` mass assignment.
  - Unescaped output in templates (`{!! !!}` in Blade, `echo` without `htmlspecialchars`).

## Swift / Kotlin (mobile)

- **Footguns:** retain cycles in closures (missing `[weak self]`), UI updates off the main thread, force unwraps (`!`) on network data, work on the main thread during scroll or launch, lifecycle leaks (observers and coroutines not tied to a lifecycle scope).

## C / C++

- **Commands:** project build (`cmake --build`, `make`), tests via `ctest` or the project's runner; sanitizers (`-fsanitize=address,undefined`) and `clang-tidy` if configured.
- **Footguns:** buffer overflows and unchecked lengths, use-after-free and double free, uninitialized variables, integer overflow and signed/unsigned mix in size calculations, missing null checks on allocation, raw `new`/`delete` where RAII or smart pointers fit, format strings built from input.

## Shell

- **Commands:** `shellcheck` on changed scripts.
- **Footguns:** missing `set -euo pipefail` (or a deliberate reason not to), unquoted variables (word splitting, globbing), `cd` without checking it succeeded, parsing `ls` output, `rm -rf "$VAR/"` with an unset `VAR`, temp files in predictable paths (`mktemp` instead), `curl | sh`.

## SQL and migrations

- **Footguns:** migrations that aren't reversible or lock large tables; data migrations mixed into schema migrations; `NOT NULL` added without a default or backfill; constraints and indexes missing for invariants the app assumes; dropping or renaming columns still read by running code (deploy order matters).

## Infrastructure as code (Terraform, Kubernetes, Docker, CI)

- **Commands:** `terraform validate`, `terraform fmt -check`, `tflint`; `kubectl apply --dry-run=client`; `hadolint` for Dockerfiles; `actionlint` for GitHub Actions.
- **Footguns:** resources replaced (destroy and create) by an innocent-looking change; secrets in plain variables or images; `latest` tags; containers running as root or `privileged`; missing resource limits; overly broad IAM; CI steps interpolating untrusted input (PR titles, branch names) into shell.
