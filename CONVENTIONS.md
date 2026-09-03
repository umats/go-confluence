# go-confluence Conventions

## Commit Messages

Use Conventional Commits.
Use `<type>(<scope>): <description>`.
Use an imperative summary after the colon.
Use `feat` for features and `fix` for defects.
Use `BREAKING CHANGE:` for incompatible public APIs.

## Git Operations

ALWAYS work on a feature branch or worktree.
NEVER push directly to `main`.
NEVER add AI attribution trailers to commits.
Run `gh pr checks` before merge when a pull request exists.

## Always Green / Shift Left

Preflight and CI MUST be green before forward work.
Run `task test && task lint && go build ./...` for Preflight.
NEVER modify `.golangci.yml`.
ALWAYS fix lint errors in the codebase.
Ignore lint errors only when absolutely necessary.
Fix defects when they cost least.
NEVER ignore a reproducible gate failure.

## Discovered Defects

Treat every reproducible gate failure as a discovered defect.
Use `quick-fix` for trivial data-only defects.
Use `fix-bug` when investigation or tests are required.
Write a bug specification when reproduction is blocked.
Commit discovered fixes separately from feature work.

## Banned Dismissive Phrases

| Phrase | Required action |
| --- | --- |
| Pre-existing | Run fix-or-log. |
| Unrelated to this session | Run fix-or-log. |
| Not introduced by my changes | Prove the regression or fix it. |
| Out of scope | Do not use this phrase to ignore a red gate. |

## Bigpowers Scripts

Use the full `~/.pi` path for bigpowers scripts.
Find scripts in the bigpowers npm installation under `~/.pi`.

## Planning Output

Write planning output only under `specs/`.
Read `specs/state.yaml` before work.
Use `specs/tech-architecture/tech-stack.md` for architecture facts.
Keep story status in `specs/execution-status.yaml`.
Keep active workflow state in `specs/state.yaml`.

## Go Code

Use standard Go formatting.
Put `context.Context` first in public network methods.
Return concrete services from constructors.
Define consumer interfaces at dependency boundaries.
Preserve wrapped sentinel errors and URL context.
Protect manual redirect validation and streaming behavior.
NEVER hand-edit `models/models_generated.go`.
Update `spec/openapi_subset.json` before regenerating models.
NEVER add dependencies without checking the standard library.

## Tests

Use table-driven `testing` and `httptest` fixtures.
Use `testify` only in tests.
Test observable behavior through public interfaces.
Add regression coverage for every defect.
Run `task test` after code changes.
Run `task test:race` for concurrency-sensitive changes.

## Defensive Code

The project uses rate limiting, retry, circuit breakers, timeouts, and graceful degradation when required.
DO use rate limits when public API behavior requires them.
DO retry only idempotent requests with bounded exponential backoff.
DO use circuit breakers around documented unstable dependencies.
ALWAYS propagate context and set network timeouts.
DO report degraded behavior explicitly to callers.

## Hard Stops

NEVER hard-code Confluence credentials or fixture IDs.
NEVER bypass redirect-host validation.
NEVER modify `vendor/`.
NEVER remove required integration-test fixtures.
