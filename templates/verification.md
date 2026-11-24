# Verification

<!--
Copy to <repo>/.claude/verification.md and fill in what applies. Delete sections you don't need.
The `verify` agent reads this before every run. Anything missing, it infers from the repo and marks as inferred.
Keep it factual and runnable: commands that work from the repo root, names that match the code.
-->

## Run

How to start the app for verification, and how to stop it.

- **Prereqs:** <!-- e.g. Node 22, Docker running, `cp .env.example .env.local` -->
- **Start:** <!-- e.g. `npm run dev` → http://localhost:5173 ; `docker compose up -d db` first -->
- **Ready when:** <!-- e.g. "GET /health returns 200", "log line: Server listening" -->
- **Seed / reset data:** <!-- e.g. `npm run db:reset && npm run db:seed` (test user: demo@example.test / demo) -->
- **Stop:** <!-- e.g. Ctrl-C, `docker compose down` -->

## Drive

How an agent exercises features, by surface. Prefer repo-provided drivers over ad-hoc scripting.

| Surface | How | Notes |
|---|---|---|
| Browser UI | <!-- Chrome DevTools MCP / Playwright / `npm run e2e -- --headed` --> | <!-- auth: log in with seed user; stable selectors use data-testid --> |
| CLI | <!-- `./bin/app <cmd>` --> | |
| HTTP API | <!-- curl against http://localhost:3000/api, token from `npm run dev:token` --> | |
| Custom driver | <!-- e.g. `scripts/drive.ts <feature> <action>` --> | |

## Feature map

The user-facing features, so agents use the same names and don't guess at the UI. Keep it short and update it when features change. For a large app, point to a separate file instead.

| Feature | Entry | Key elements | States to check |
|---|---|---|---|
| <!-- Reports: CSV export --> | <!-- /reports → "Export" button --> | <!-- [data-testid=export-btn], toast --> | <!-- empty report, >10k rows, export fails --> |

## Checks

The deterministic checks this repo has. List anything that gives a pass/fail without an LLM's judgment: unit and integration suites, e2e, property-based tests, fuzzers, model checkers (e.g. TLA+), proofs (e.g. Lean), visual regression, type checks, linters, statechart or schema validation.

| Name | Command | Covers (paths/globs) | Cost | Run when |
|---|---|---|---|---|
| <!-- unit --> | <!-- `npm test` --> | <!-- `src/**` --> | <!-- ~30s --> | <!-- always --> |
| <!-- e2e --> | <!-- `npm run e2e` --> | <!-- `src/ui/**` --> | <!-- ~4m --> | <!-- UI changed --> |
| <!-- sync-model --> | <!-- `tlc specs/Sync.tla` --> | <!-- `src/sync/**` --> | <!-- ~10m --> | <!-- covered paths changed --> |
| <!-- props --> | <!-- `npm run test:props -- --runs 1000` --> | <!-- `src/editor/**` --> | <!-- ~2m --> | <!-- covered paths changed --> |

**Run when:** `always`, `<surface> changed`, `covered paths changed`, or `on request`. Expensive checks should not be `always`.

## Artifacts

- **Location:** <!-- default: outside the repo, given by the conductor. Or e.g. `.verify/` (gitignored) -->
- **Capture:** <!-- e.g. screenshot per UI criterion, recording for multi-step flows, HAR on network failures -->
- **Diagrams:** <!-- e.g. `npm run statechart -- src/sync/machine.ts > $ARTIFACTS/sync.svg` for state-machine changes -->

## Off limits

Things verification must never do. The agent treats this section as absolute.

- <!-- Never point at production or staging URLs/DBs -->
- <!-- Never trigger real payments, emails, SMS, or webhooks to third parties -->
- <!-- Never run `db:reset` against anything but the local dev DB -->
