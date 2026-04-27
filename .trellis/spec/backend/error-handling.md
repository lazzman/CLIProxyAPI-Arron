# Error Handling

> How errors are wrapped, logged, and returned in the current Go/Gin codebase.

---

## Overview

This repository uses two complementary error-handling styles:

- Internal package errors are returned as contextual Go errors, usually wrapped
  with `%w` and prefixed by subsystem name.
- HTTP responses are shaped according to the endpoint family:
  - OpenAI-compatible endpoints return OpenAI-style payloads.
  - Management/private endpoints often return `gin.H` JSON bodies.

The codebase does not centralize all errors into one custom error hierarchy.
Instead, each subsystem adds enough operation context for logs and callers to
decide whether an error is fatal, retryable, or safe to downgrade to a warning.

---

## Error Types And Shapes

### Wrapped Go errors with subsystem prefixes

This is the dominant internal pattern.

Real examples:

- `fmt.Errorf("postgres store: open database connection: %w", err)`
- `fmt.Errorf("postgres store: write auth file: %w", err)`
- `fmt.Errorf("object store: bucket is required")`

See `internal/store/postgresstore.go` and `internal/store/objectstore.go`.

### OpenAI-compatible error payloads for public AI APIs

Shared handler code builds provider-compatible JSON error bodies instead of
returning ad hoc JSON.

Real example:

- `sdk/api/handlers/handlers.go` defines `ErrorResponse`, `ErrorDetail`, and
  `BuildErrorResponseBody(...)`.

`BuildErrorResponseBody` maps HTTP status codes into API error types such as:

- `authentication_error`
- `permission_error`
- `rate_limit_error`
- `server_error`

### Simple `gin.H` payloads for management endpoints

Management routes do not try to look like OpenAI APIs. They return small,
direct JSON bodies that match the private management surface.

Real example:

- `internal/api/handlers/management/handler.go`

---

## Error Handling Patterns

### Return context-rich errors from storage and runtime code

- Add the subsystem prefix to every returned error.
- Include the failed operation, not just the lower-level error string.
- Use `%w` when a caller may need `errors.Is` or `errors.As`.

Good examples:

- `cmd/server/main.go` downgrades missing `.env` to a non-fatal path by checking
  `errors.Is(errLoad, os.ErrNotExist)`.
- `internal/store/postgresstore.go` wraps path, JSON, file, and SQL failures
  with operation-specific messages.

### Downgrade expected non-fatal failures to warnings

The repository does not treat every bad record as startup-fatal.

Real examples:

- Invalid mirrored auth rows are skipped with warnings in
  `internal/store/postgresstore.go`.
- `.env` load errors are logged as warnings only when the file exists but cannot
  be parsed or read in `cmd/server/main.go`.

### Recover panics at the Gin boundary

- `internal/logging/gin_logger.go` provides `GinLogrusRecovery()`.
- Regular panics are logged with stack trace and return HTTP 500.
- `http.ErrAbortHandler` is re-panicked intentionally so `net/http` can abort
  the connection without an extra noisy recovery log.

Tests for that behavior live in `internal/logging/gin_logger_test.go`.

---

## API Error Responses

### Public AI endpoints

Use OpenAI-compatible error payloads when the route belongs to the public API
surface under `/v1`, `/v1beta`, or equivalent provider-compat layers.

Real examples:

- `sdk/api/handlers/openai/openai_handlers.go` returns
  `handlers.ErrorResponse{...}` for invalid request bodies.
- Shared helpers in `sdk/api/handlers/handlers.go` preserve upstream JSON error
  payloads when they are already valid JSON.

### Management endpoints

Use straightforward management-specific JSON via `AbortWithStatusJSON` or
`JSON(...)` with `gin.H` or small structs.

Real examples:

- `"remote management disabled"`
- `"remote management key not set"`

See `internal/api/handlers/management/handler.go`.

---

## Common Mistakes

- Do not return bare `err` from store/runtime code when the current package can
  add operation context.
- Do not use the OpenAI error envelope for management-only endpoints.
- Do not return ad hoc `gin.H{"error": ...}` payloads from public compatibility
  endpoints when shared handler helpers already define the expected shape.
- Do not convert expected absence (`os.ErrNotExist`, empty local files, missing
  DB rows during bootstrap) into fatal startup failures unless the existing flow
  already requires it.
- Do not swallow panics inside deep helper code. Panic recovery belongs at the
  Gin middleware boundary.
