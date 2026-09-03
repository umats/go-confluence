# Project Context

## Scope and Evidence

Cold map completed from the Go module, source packages, generated OpenAPI subset, tests, CI, and task configuration. The repository is a Go **library**, not a Confluence server, web application, or production CLI. The only executable (`cmd/discover_ids`) is build-ignored and intended for local fixture discovery.

## Stack

- **Language/runtime:** Go `1.27` (`go.mod`); a single module, `github.com/umats/go-confluence`.
- **Protocol/runtime dependencies:** Go standard library only in production: `net/http`, `net/url`, `encoding/json`, `context`, and `io`.
- **Direct third-party dependency:** `github.com/stretchr/testify v1.12.1`, used only by tests. Its indirect dependency is `go.yaml.in/yaml/v3 v3.0.5`.
- **API contract:** Confluence REST v2 plus legacy/concrete endpoints required for PDF and attachment downloads. `spec/openapi_subset.json` supplies the contract subset; `models/models_generated.go` is generated and must not be edited manually.
- **Tooling:** Task v3 (`Taskfile.yml`), `golangci-lint` v2.12.1 in CI, and `gomarkdoc` for per-package `DOCS.md` generation.
- **CI:** GitHub Actions runs unit tests, race tests, lint, and credentialed integration tests for same-repository PRs/main pushes.

No ORM, datastore, HTTP framework, authentication SDK, UI, metrics library, or external logging framework is present or needed for this client library.

## Architecture

### Package boundaries

| Area | Responsibility |
| --- | --- |
| Root `confluence` | Public `Client`, functional options, public service interfaces, compatibility aliases, and service accessors. |
| `internal/transport` | Shared request construction/basic auth, base URL resolution, REST v2 JSON execution, non-2xx `APIError`, and reflection-based query encoding. |
| `page`, `space`, `attachment`, `label` | Domain-named REST v2 service packages. Each holds a `*transport.V2Client` and translates typed method calls into endpoint paths and `DoJSON` calls. |
| `export` | PDF workflow state machine for Server/Data Center redirects and Cloud background tasks/polling. |
| `params` | Typed query structs and generic paginated result wrapper aliases. |
| `models` | Generated request/response types and string enums. |
| `cmd/discover_ids` | Build-ignored local helper to enumerate live fixture IDs; not part of the shipped library. |

This is a compact, domain-package client with a shared transport layer—not controller/service/repository layering and not hexagonal architecture. Root accessors return interfaces for consumer-side dependency injection while concrete `*Service` implementations remain in their domain packages.

### Primary flows

**REST v2:**

```text
caller → confluence.NewClient(options) → Client.Page/Space/Attachment/Label()
       → domain Service → transport.BuildQuery + V2Client.DoJSON
       → URL /wiki/api/v2/... + Basic Auth → http.Client → typed JSON result/APIError
```

`NewClient` validates and normalizes the base URL, sets a 30-second HTTP timeout, and disables automatic redirects. `transport.NewV2URL`/`V2Client.newURL` add `/wiki/api/v2` unless the configured base path already ends in `/wiki`.

**PDF export:**

```text
Export.PageTo → export action
  302 → validate redirect host → stream PDF
  200 application/pdf → stream PDF
  200 HTML → extract task ID → poll progress endpoint → stream result PDF
```

Cloud polling honors the caller context and optional `WithPollTimeout`; it waits with a timer between requests. `Page` is the intentional in-memory wrapper over streaming `PageTo`.

**Attachment download:**

```text
Attachment.Download → GET attachment metadata → choose page/blog/custom-content container
  → supported legacy attachment-download endpoint → validated media redirect → stream to io.Writer
```

Direct URL download remains deprecated. This workflow intentionally avoids the metadata `downloadLink` fallback to prevent ambiguous/partial-stream behavior.

## Conventions (Observed)

### API and types

- Public network methods take `context.Context` first and return typed generated models, typed generic list wrappers, or `error`.
- REST is JSON over `net/http`; JSON field names and query keys follow the Confluence contract (mostly kebab-case such as `body-format`, with contract-defined exceptions such as `mediaType`).
- Pagination uses `params.MultiEntityResult[T]` with `results` and `_links`.
- Query parameters are pointer-heavy to preserve optionality, with slices repeated as query values. `BuildQuery` accepts tagged structs, `url.Values`, and two string-keyed map forms.
- Models and root aliases are public compatibility surface. Change them deliberately; the root package preserves aliases to `params` and `models` types.
- The only production `any` uses are the generic JSON/query boundary (`DoJSON`, `BuildQuery`); no `unsafe` is used.

### Error handling and resource management

- Errors are returned and wrapped at each operation boundary with contextual verbs and, for export/download paths, attempted URLs.
- REST non-2xx responses become typed `transport.APIError{StatusCode, Body}`. Export-specific sentinel errors (`ErrMissingLocation`, `ErrTaskIDNotFound`, `ErrTaskFailed`, `ErrTaskResultEmpty`) are re-exported from the root package and retained through wrapping for `errors.Is`.
- Request/response bodies are closed locally; large PDFs and attachments stream via `io.Copy` to a caller-owned `io.Writer`.
- Redirects are disabled globally so application code can validate `Location` hosts. The base host is allowlisted by default; callers may opt in to more redirect hosts or cross-host custom-content URLs. HTTPS is opt-in through `WithRequireHTTPS`.

### Testing and quality

- Unit tests use table-driven `testing`, `httptest`, custom `RoundTripper`s/readers/writers, plus `testify` assertions. They validate method/path/query/auth/body shape, JSON errors, redirect safety, streaming failures, cancellation, and sentinel errors.
- One fuzz target covers Cloud task-ID extraction. Integration tests are guarded by the `integration` build tag, use environment-provided live Confluence credentials/fixture IDs, and fail when required values are absent.
- Current cold baseline: `go test ./...` passed **152 tests across 10 packages**.
- Production packages have no request logging, metrics, tracing, or health endpoint. This is appropriate for an embeddable client; callers own observability around API calls. The deprecated direct-download API uses `slog.Default()` solely to emit its deprecation warning.

## Signals / Active Considerations

1. **Shared transport is the highest-blast-radius seam.** `internal/transport` is used by every service. Changes to URL joining, query reflection, JSON decoding, auth, or `APIError` require full unit/integration coverage and compatibility review.
2. **Generated-model boundary.** Update `spec/openapi_subset.json` and regenerate `models/models_generated.go`; never hand-edit the generated file. New endpoints normally require changes across spec/models, params, service interface, service implementation, root aliases, tests, and docs.
3. **Export and attachment downloads are security-critical.** Preserve manual redirects, host allowlists, context propagation, URL-rich errors, and streaming behavior. Do not replace them with automatic redirect following.
4. **Two platform variants drive complexity.** Server/Data Center PDF export uses redirects; Cloud export uses HTML task discovery plus polling. Changes must preserve both flows.
5. **Intentional public compatibility surface.** `services_interfaces.go` composes narrower read/write interfaces, and `services_compat.go` retains root aliases. Avoid moving/removing exported identifiers casually in this published module (latest tag observed: `v0.3.0`).
6. **Observed documentation drift:** the README quick-start calls `Page().Get` with a string, but the public method accepts an `int64`; integration tests parse `CONFLUENCE_PAGE_ID` to `int64`. Correct the example before relying on it as compilable user guidance.
7. **Observed deprecation-warning defect:** `defaultDownloadByURLDeprecationWarningState` constructs a new warning on each call, so the intended once-only warning state is not shared and the reset helper cannot affect future calls. Add a focused behavioral test and stabilize the state only if preserving one warning per process is required.
8. **Minor duplication at the transport seam:** `Client.NewV2URL` and `V2Client.newURL` implement the same `/wiki/api/v2` URL construction, while services use the latter. Consolidate only when touching that behavior; redirect/URL handling is high-risk and this small duplication is currently contained.
9. **No retry/backoff/rate-limit policy exists.** Requests use the configured client timeout; Cloud polling has a fixed interval. Add retries only for a documented Confluence failure mode and with idempotency rules, not as a generic transport abstraction.
