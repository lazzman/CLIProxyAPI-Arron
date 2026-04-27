# Logging Guidelines

> Logging conventions implemented by the shared `logrus`-based infrastructure.

---

## Overview

This repository uses `github.com/sirupsen/logrus` for process logs and a custom
request-log subsystem for detailed HTTP request/response capture.

Primary logging entrypoints:

- Global logger setup: `internal/logging/global_logger.go`
- Gin request logging and recovery middleware: `internal/logging/gin_logger.go`
- Detailed request log files: `internal/logging/request_logger.go`
- Public SDK re-export for request logging: `sdk/logging/request_logger.go`

The codebase distinguishes between:

- Process logs: stdout or rotating `main.log`
- Request summary logs: one line per request through Gin middleware
- Request detail logs: file-based request/response dumps when enabled

---

## Log Levels

### `info`

Use for normal lifecycle and successful request flow.

Real examples:

- Successful storage backend activation in `cmd/server/main.go`
- Normal HTTP request summaries for non-error responses in
  `internal/logging/gin_logger.go`

### `warn`

Use for non-fatal issues where the process can continue safely.

Real examples:

- Failed `.env` load when the file exists but cannot be loaded:
  `cmd/server/main.go`
- Skipping malformed persisted auth data:
  `internal/store/postgresstore.go`
- Failed cleanup of temporary or log resources:
  `internal/logging/request_logger.go`

### `error`

Use for startup failures, panic recovery, and 5xx request summaries.

Real examples:

- Storage backend bootstrap failure in `cmd/server/main.go`
- Recovered panic with stack trace in `internal/logging/gin_logger.go`
- Request summaries for HTTP 5xx responses in `internal/logging/gin_logger.go`

---

## Structured Logging

### Shared formatter

`internal/logging/global_logger.go` installs a custom formatter with:

- timestamp
- request ID
- log level
- caller file and line when available
- selected ordered fields

Current display shape:

```text
[2025-12-23 20:14:04] [a1b2c3d4] [info ] [manager.go:524] message field=value
```

### Request IDs are selective, not universal

- Request IDs are generated only for AI API paths such as `/v1/chat/completions`
  and `/v1/responses`.
- Non-AI paths log the placeholder request ID `--------`.

Real implementation:

- `internal/logging/gin_logger.go`

### Request summary logs are status-driven

- 5xx => `Error`
- 4xx => `Warn`
- everything else => `Info`

The request summary line includes:

- status code
- latency
- client IP
- method
- masked path and query string
- private Gin error string when present

### Request detail logs are file-based and opt-in

- `internal/logging/request_logger.go` writes detailed request/response logs
  under the resolved logs directory.
- `internal/api/middleware/request_logging.go` decides when body capture is
  allowed and skips management endpoints entirely.
- When full request logging is disabled, error-only capture still allows small
  known-size request bodies for diagnosis.

---

## What To Log

- Startup and backend selection decisions:
  `cmd/server/main.go`
- Storage sync anomalies that are survivable:
  `internal/store/postgresstore.go`
- Recovered panics with enough context to diagnose them:
  `internal/logging/gin_logger.go`
- Request summaries and optional detailed request/response artifacts:
  `internal/logging/gin_logger.go`, `internal/logging/request_logger.go`
- Aggregate usage metrics through the statistics plugin:
  `internal/usage/logger_plugin.go`

## Codex Usage 统计发布约束

- `usageReporter.finalize(...)` 会在没有错误时补发一条成功统计，这个语义只适合“返回前已经拿到最终 usage”的 provider。
- Codex `/responses` 的 HTTP SSE 与 WebSocket 路径必须等到终态事件里的 `response.usage` 被解析后，才允许发布成功统计。
- 如果 `response.completed` 或 `response.done` 缺失 `response.usage`，不要补发伪造的 `0 token` 成功统计。
- Codex prompt cache 与请求会话绑定时，HTTP 请求只把 `Session_id` 绑定到 `prompt_cache_key`；WebSocket 的 `Conversation_id` 可以随 prompt cache 透传，但不要在 transport 级默认逻辑之前强行注入同值 `Session_id`。

---

## What NOT To Log

- Unmasked sensitive query values. Use the existing masking path:
  `util.MaskSensitiveQuery(...)`
- Management endpoint request bodies or secrets. The request logging middleware
  explicitly skips `/v0/management` and `/management`.
- Raw credentials or auth payload dumps in process logs.
- Extra ad hoc log formats that bypass the shared formatter unless a task has a
  very specific reason.

Even when detailed request logging is enabled, preserve the existing guardrails:

- most plain GET requests are skipped
- management routes are skipped
- request body capture is size-limited in error-only mode

See `internal/api/middleware/request_logging.go`.
