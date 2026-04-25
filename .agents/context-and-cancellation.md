# Context and Cancellation in LocalAI

This guide covers how to properly use Go's `context.Context` for request lifecycle management, cancellation propagation, and timeout handling across LocalAI backends and API handlers.

## Overview

LocalAI uses `context.Context` throughout the request pipeline to:
- Propagate cancellation when clients disconnect
- Enforce per-request timeouts
- Pass request-scoped values (trace IDs, auth info) to backends
- Clean up resources when inference is interrupted

## Basic Pattern

All handler functions and backend calls should accept and forward a context:

```go
func (h *Handler) ChatCompletion(c *fiber.Ctx) error {
    // Derive a context from the fiber request context
    ctx, cancel := context.WithTimeout(
        c.Context(),
        h.cfg.InferenceTimeout,
    )
    defer cancel()

    result, err := h.backend.Infer(ctx, req)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            return fiber.NewError(fiber.StatusGatewayTimeout, "inference timed out")
        }
        if errors.Is(err, context.Canceled) {
            // Client disconnected — log and return without error response
            log.Debug().Msg("client disconnected, inference cancelled")
            return nil
        }
        return err
    }
    return c.JSON(result)
}
```

## Fiber Context Adapter

Fiber uses `fasthttp` under the hood, which has its own context. Use the helper below to bridge to a standard `context.Context`:

```go
// FiberCtx returns a standard context.Context that is cancelled
// when the underlying fasthttp request is done or the client disconnects.
func FiberCtx(c *fiber.Ctx) context.Context {
    return c.Context()
}
```

For long-running operations, prefer creating a child context with an explicit timeout rather than relying solely on the fiber context.

## Timeout Hierarchy

```
Server-level timeout (read/write deadlines)
  └── Request-level timeout (per handler, from config)
        └── Backend-level timeout (model inference, from model config)
              └── Sub-operation timeout (tokenisation, sampling, etc.)
```

Each layer should be *shorter* than or equal to its parent.

## Passing Values via Context

Use typed keys to avoid collisions:

```go
type contextKey string

const (
    ContextKeyRequestID contextKey = "request_id"
    ContextKeyModelName contextKey = "model_name"
    ContextKeyAPIKey    contextKey = "api_key"
)

// WithRequestID attaches a request ID to the context.
func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, ContextKeyRequestID, id)
}

// RequestIDFromCtx retrieves the request ID, returning empty string if absent.
func RequestIDFromCtx(ctx context.Context) string {
    v, _ := ctx.Value(ContextKeyRequestID).(string)
    return v
}
```

Inject these in middleware so all downstream code can read them:

```go
func RequestIDMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        id := c.Get("X-Request-ID")
        if id == "" {
            id = uuid.New().String()
        }
        // Store in fiber locals for fiber-aware code
        c.Locals(string(ContextKeyRequestID), id)
        c.Set("X-Request-ID", id)
        return c.Next()
    }
}
```

## Cancellation in Streaming Responses

When streaming tokens, check for cancellation between each token to avoid wasting compute after a client disconnects:

```go
func streamTokens(ctx context.Context, w *StreamWriter, tokens <-chan string) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case tok, ok := <-tokens:
            if !ok {
                return nil // stream finished normally
            }
            if err := w.WriteToken(tok); err != nil {
                return fmt.Errorf("write token: %w", err)
            }
        }
    }
}
```

## Backend Cancellation

Backends that call into C/C++ libraries (e.g. llama.cpp) may not natively support Go context cancellation. Use a goroutine + done channel pattern:

```go
func (b *LlamaCppBackend) Infer(ctx context.Context, req *InferRequest) (*InferResponse, error) {
    type result struct {
        resp *InferResponse
        err  error
    }
    ch := make(chan result, 1)

    go func() {
        resp, err := b.llamaInferBlocking(req) // blocking C call
        ch <- result{resp, err}
    }()

    select {
    case <-ctx.Done():
        b.requestCancel() // signal llama.cpp to abort via its own API
        return nil, ctx.Err()
    case r := <-ch:
        return r.resp, r.err
    }
}
```

## Configuration

Timeouts are sourced from the model config and global server config:

```yaml
# Model-level config (models/my-model.yaml)
timeout: 120s          # per-request inference timeout
stream_timeout: 300s   # extended timeout for streaming requests
```

```go
// Resolved in the handler
timeout := h.modelCfg.Timeout
if req.Stream && h.modelCfg.StreamTimeout > 0 {
    timeout = h.modelCfg.StreamTimeout
}
ctx, cancel := context.WithTimeout(baseCtx, timeout)
defer cancel()
```

## Testing Cancellation

```go
func TestInferCancellation(t *testing.T) {
    backend := newMockBackend(50 * time.Millisecond) // simulates slow inference
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
    defer cancel()

    _, err := backend.Infer(ctx, &InferRequest{Prompt: "hello"})
    require.ErrorIs(t, err, context.DeadlineExceeded)
}
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Ignoring `ctx.Done()` in a hot loop | Add a `select` with `ctx.Done()` case |
| Passing `context.Background()` through a handler | Thread the request context from `c.Context()` |
| Not calling `cancel()` after `WithTimeout` | Always `defer cancel()` immediately after creation |
| Using `context.WithValue` with built-in types as keys | Define a private `contextKey` type |
