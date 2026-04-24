# Request Lifecycle in LocalAI

This document describes how a request flows through LocalAI from the HTTP layer down to the backend model and back.

## Overview

```
Client Request
    │
    ▼
[Fiber HTTP Server]
    │
    ▼
[Auth Middleware]  ←── api-key validation, see middleware-and-security.md
    │
    ▼
[Request Logger]  ←── structured logging, see observability-and-logging.md
    │
    ▼
[Route Handler]   ←── e.g. /v1/chat/completions, /v1/completions
    │
    ▼
[Request Validation]  ←── schema validation, error normalization
    │
    ▼
[Model Resolution]    ←── resolve model name → config file
    │
    ▼
[Backend Dispatch]    ←── select gRPC backend (llama.cpp, whisper, etc.)
    │
    ▼
[gRPC Backend Call]   ←── forward request to backend process
    │
    ▼
[Response Streaming / Aggregation]
    │
    ▼
Client Response
```

## 1. HTTP Server Initialization

LocalAI uses the [Fiber](https://gofiber.io/) web framework. Routes are registered
during startup in `api/api.go`. Each feature area registers its own routes via
a `RegisterXxxRoutes(app, cl, ml, ...)` function — see `api-endpoints-and-auth.md`
for the pattern.

```go
// Typical startup sequence (simplified)
app := fiber.New(fiber.Config{
    BodyLimit: cfg.UploadLimit,
})
app.Use(middleware.RequestLogger())
app.Use(middleware.AuthMiddleware(apiKeys))
routes.RegisterChatRoutes(app, cl, ml, appConfig)
routes.RegisterCompletionRoutes(app, cl, ml, appConfig)
// … other route groups
app.Listen(addr)
```

## 2. Auth Middleware

Every request passes through `AuthMiddleware` before reaching a handler.
See `.agents/middleware-and-security.md` for implementation details.

- Reads `Authorization: Bearer <key>` or `x-api-key: <key>` header.
- If `LOCALAI_API_KEY` is unset, all requests are allowed (open mode).
- On failure returns `401 Unauthorized` with a JSON error body.

## 3. Route Handler Entry Point

Handlers follow the pattern described in `api-endpoints-and-auth.md`:

```go
func ChatEndpoint(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        // 1. Parse & validate request body
        // 2. Resolve model config
        // 3. Call backend
        // 4. Stream or return response
    }
}
```

## 4. Model Resolution

The model name from the request (e.g. `"gpt-4"`) is resolved to a
`BackendConfig` struct:

1. Look up `<model-name>.yaml` in the models directory.
2. Fall back to auto-configuration if no YAML exists.
3. The config specifies which backend binary to use (`llama-cpp`, `whisper`, etc.),
   context size, GPU layers, template paths, and so on.

See `.agents/model-configuration.md` for the full config schema.

## 5. Backend Dispatch

LocalAI communicates with model backends over **gRPC**. Each backend is a
separate process (or shared process for concurrent requests).

```
Handler → backends.ModelInference(ctx, request, ml, config)
              │
              ├─ ensure backend process is running  (LoadModel)
              ├─ build gRPC request proto
              └─ call backend.Predict / backend.PredictStream
```

`LoadModel` (see `.agents/debugging-backends.md`) starts the backend subprocess
if it is not already alive and returns a gRPC client stub.

## 6. Streaming vs. Aggregated Responses

### Streaming (SSE)

When `stream: true` is set in the request:

```go
c.Set("Content-Type", "text/event-stream")
c.Set("Cache-Control", "no-cache")
c.Context().SetBodyStreamWriter(func(w *bufio.Writer) {
    for token := range tokenChan {
        fmt.Fprintf(w, "data: %s\n\n", token)
        w.Flush()
    }
    fmt.Fprint(w, "data: [DONE]\n\n")
})
```

### Non-Streaming

The handler waits for the full response, then returns a single JSON object
compatible with the OpenAI API schema.

## 7. Error Handling

All errors are normalized through `errorResponse` (see `.agents/error-handling.md`)
before being sent to the client. Panics inside handlers are recovered by
`safeCall` and converted to `500 Internal Server Error` responses.

## 8. Observability

- Request duration, model name, status code, and token counts are logged
  at the end of each request via the `RequestLogger` middleware.
- Prometheus metrics are incremented for inference latency and request counts.
- See `.agents/observability-and-logging.md` for details.

## Key Files

| File | Purpose |
|------|---------|
| `api/api.go` | Server setup, route registration |
| `api/middleware/` | Auth, logging, recovery middleware |
| `api/routes/` | Individual route handlers |
| `pkg/model/loader.go` | Model loading & backend lifecycle |
| `pkg/backends/` | gRPC client wrappers per backend |
| `pkg/config/` | BackendConfig, ApplicationConfig |
