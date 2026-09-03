# go-confluence — Claude Code

Read CONVENTIONS.md before GitHub or git operations.

<!-- BEGIN bigpowers:context-routing -->
## Context Routing

Read `AGENTS.md` for repository-specific implementation guidance.
Read `specs/` before planning or implementing work.
<!-- END bigpowers:context-routing -->

<!-- BEGIN bigpowers:learned-preferences -->
## Learned User Preferences

- Use Conventional Commits.

## Workspace Facts

- Go 1.27 is the project toolchain.
<!-- END bigpowers:learned-preferences -->

<!-- BEGIN bigpowers:project -->
## Project

go-confluence is a Go client library for Confluence REST v2 APIs and PDF export workflows.

Stack: Go, `net/http`, and Go modules.

## Commands

| Action | Command |
| --- | --- |
| Run | N/A; this repository ships no executable. |
| Test | `task test` |
| Build | `go build ./...` |
| Lint | `task lint` |
| Preflight | `task test && task lint && go build ./...` |
| CI | `gh pr checks` |

## Architecture

The root client exposes domain services through `internal/transport`.
The export package handles redirects and Cloud task polling.

## Conventions

- Use domain packages and functional options.
- Put `context.Context` first in network methods.
- Use typed `params` and generated `models` values.
- Stream PDFs and attachments through `io.Writer`.

## Never

- NEVER hand-edit `models/models_generated.go`.
- NEVER hard-code credentials or fixture IDs.
- NEVER bypass redirect-host validation.
- NEVER modify `vendor/`.

## Agent Rules

- MUST use bigpowers skills for feature and bug work.
- MUST write planning output under `specs/`.
- MUST keep Preflight and CI green.
- MUST run focused tests after each code change.
- DO write the smallest correct change.
<!-- END bigpowers:project -->
