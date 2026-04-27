# Quality Guidelines

> Current backend quality gates and review expectations for this repository.

---

## Overview

This repository relies more on code review, targeted Go tests, and buildability
than on a large committed lint stack.

What exists today:

- CI pull requests run a server build in `.github/workflows/pr-test-build.yml`
- Many packages have colocated `*_test.go` coverage
- Some higher-level integration tests live under `test/`
- `AGENTS.md` adds project-wide expectations such as searching existing code
  first, keeping comments focused on why, and verifying work before reporting it

What does not exist in the repository today:

- no committed `golangci-lint` configuration
- no committed custom lint script
- no standalone migration/test harness just for storage backends

Document reality when writing or reviewing code: build and targeted tests matter
more here than satisfying an imagined lint config.

---

## Forbidden Patterns

- Do not modify `internal/translator/**` casually. PRs that touch that path are
  blocked by `.github/workflows/pr-path-guard.yml`.
- Do not add comments that only restate the code. Follow the project rule from
  `AGENTS.md`: comments should explain why, trade-offs, or non-obvious behavior.
- Do not put new app-private implementation in `sdk/` just because it is
  convenient, and do not hide reusable public surface under `internal/`.
- Do not bypass the existing logging/error helpers with one-off response or log
  shapes when a shared package already owns that behavior.
- Do not claim a lint or test requirement in docs unless it really exists in the
  repository.

---

## Required Patterns

- Search the existing code and adjacent tests before introducing a new pattern.
- Keep tests close to the package unless the scenario truly spans packages.
- Preserve endpoint-family consistency:
  - OpenAI-compatible routes should return compatibility-friendly error shapes.
  - Management routes should keep their simpler private JSON responses.
- Preserve storage invariants when touching persistence code:
  - local spool files still matter
  - path normalization and directory-boundary checks stay intact
- Keep `cmd/server` buildable after backend changes because that is the primary
  CI gate on pull requests.

---

## Testing Requirements

### Use targeted package tests for the area you changed

Common examples already in the repo:

- `internal/logging/gin_logger_test.go`
- `internal/api/server_test.go`
- `sdk/api/handlers/openai/openai_responses_handlers_stream_test.go`
- `sdk/cliproxy/service_reload_test.go`
- `test/amp_management_test.go`

When adding or changing behavior:

- prefer adding a `*_test.go` file beside the changed package
- use top-level `test/` only for broader integration-style behavior
- run at least the affected package tests locally
- run a broader build or test command when touching shared startup/runtime code

### Build remains a first-class gate

Current PR CI runs:

```text
go build -o test-output ./cmd/server
```

from `.github/workflows/pr-test-build.yml`.

For backend work, treat `go build ./cmd/server` or the equivalent command above
as mandatory verification whenever code changes could affect server startup.

---

## Code Review Checklist

- Is the code in the correct layer (`cmd/`, `internal/`, `sdk/`, `test/`)?
- If persistence changed, did the code preserve spool mirroring, safe paths, and
  context-aware SQL/object-store calls?
- If HTTP behavior changed, does the response format still match the endpoint
  family that owns the route?
- If logging changed, are secrets still masked and management routes still
  excluded from request-body logging?
- Is there package-local test coverage for the new or changed behavior?
- Does the server still build, and were the touched package tests run?
