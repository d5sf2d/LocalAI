# Error Handling in LocalAI

This guide covers best practices for error handling, custom error types, and error propagation patterns used throughout the LocalAI codebase.

## Core Principles

1. **Wrap errors with context** — always add meaningful context when propagating errors up the call stack.
2. **Use typed errors** — define sentinel errors or custom error types for conditions callers need to handle.
3. **Log at the boundary** — log errors where they are handled, not where they originate.
4. **Never swallow errors silently** — if an error is intentionally ignored, document why.

## Standard Error Wrapping

Use `fmt.Errorf` with `%w` to wrap errors so callers can use `errors.Is` / `errors.As`:

```go
if err != nil {
    return fmt.Errorf("loading model %q: %w", modelName, err)
}
```

## Sentinel Errors

Define package-level sentinel errors for conditions that callers must distinguish:

```go
var (
    ErrModelNotFound   = errors.New("model not found")
    ErrBackendNotReady = errors.New("backend not ready")
    ErrInvalidConfig   = errors.New("invalid model configuration")
)
```

Usage:

```go
if errors.Is(err, ErrModelNotFound) {
    c.Status(fiber.StatusNotFound)
    return c.JSON(schema.ErrorResponse{Error: err.Error()})
}
```

## Custom Error Types

When you need to carry additional data with an error, define a struct that implements the `error` interface:

```go
// ValidationError carries field-level detail for config validation failures.
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on field %q: %s", e.Field, e.Message)
}
```

Callers can extract the detail:

```go
var ve *ValidationError
if errors.As(err, &ve) {
    log.Warn().Str("field", ve.Field).Msg(ve.Message)
}
```

## HTTP Error Responses

All API handlers should return a consistent JSON error body. Use the shared `schema.ErrorResponse` type:

```go
func errorResponse(c *fiber.Ctx, status int, err error) error {
    log.Error().Err(err).Int("status", status).Msg("request error")
    return c.Status(status).JSON(schema.ErrorResponse{
        Error: &schema.APIError{
            Message: err.Error(),
            Code:    status,
        },
    })
}
```

Common mappings:

| Condition              | HTTP Status                  |
|------------------------|------------------------------|
| Model not found        | 404 Not Found                |
| Invalid request body   | 400 Bad Request              |
| Backend not ready      | 503 Service Unavailable      |
| Internal / unexpected  | 500 Internal Server Error    |

## Backend / gRPC Errors

When a backend call fails, inspect the gRPC status code before deciding how to surface the error:

```go
import "google.golang.org/grpc/status"

st, ok := status.FromError(err)
if ok {
    switch st.Code() {
    case codes.Unavailable:
        return fmt.Errorf("%w: %s", ErrBackendNotReady, st.Message())
    case codes.InvalidArgument:
        return fmt.Errorf("%w: %s", ErrInvalidConfig, st.Message())
    }
}
return fmt.Errorf("backend call failed: %w", err)
```

## Panic Recovery

Backend goroutines should recover from panics and convert them to errors so the rest of the service stays alive:

```go
func safeCall(fn func() error) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recovered: %v", r)
            log.Error().Msgf("panic in backend call: %v", r)
        }
    }()
    return fn()
}
```

## Logging Errors

Use the zerolog helpers from `observability-and-logging.md`. Always attach the error object:

```go
log.Error().Err(err).Str("model", modelName).Msg("failed to load model")
```

Avoid `log.Error().Msg(err.Error())` — it loses the structured `error` field.

## Checklist

- [ ] Errors are wrapped with `%w` and contextual message
- [ ] Sentinel / typed errors used for distinguishable conditions
- [ ] HTTP handlers return `schema.ErrorResponse` JSON
- [ ] gRPC status codes mapped to domain errors
- [ ] Panics recovered in long-running goroutines
- [ ] Errors logged with `.Err(err)` at the handling site
