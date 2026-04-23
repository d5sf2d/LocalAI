# Observability and Logging in LocalAI

This document covers logging practices, metrics, tracing, and general observability patterns used throughout LocalAI.

## Logging

LocalAI uses [zerolog](https://github.com/rs/zerolog) for structured logging. The logger is initialized in `pkg/utils/logging.go` and available globally.

### Import

```go
import "github.com/rs/zerolog/log"
```

### Log Levels

| Level | Usage |
|-------|-------|
| `log.Trace()` | Very verbose, per-token or per-request detail |
| `log.Debug()` | Development/diagnostic info |
| `log.Info()` | Normal operational events |
| `log.Warn()` | Recoverable issues, degraded behavior |
| `log.Error()` | Errors that affect a single request |
| `log.Fatal()` | Unrecoverable startup errors only |

### Structured Fields

Always prefer structured fields over string interpolation:

```go
// Good
log.Info().
    Str("model", modelName).
    Str("backend", backendName).
    Int("tokens", tokenCount).
    Msg("inference completed")

// Avoid
log.Info().Msgf("inference completed for model %s using backend %s", modelName, backendName)
```

### Common Field Names

Use consistent field names across the codebase:

- `model` — model name or path
- `backend` — backend identifier (e.g., `llama-cpp`, `whisper`)
- `request_id` — unique request identifier
- `duration_ms` — elapsed time in milliseconds
- `error` — error string (use `.Err(err)` shorthand)
- `endpoint` — HTTP route path
- `gallery` — gallery model name

### Request-Scoped Logging

For HTTP handlers, attach request context to log entries:

```go
func MyHandler(c *fiber.Ctx) error {
    requestID := c.Get("X-Request-ID", uuid.New().String())
    logger := log.With().
        Str("request_id", requestID).
        Str("endpoint", c.Path()).
        Logger()

    logger.Debug().Msg("handling request")
    // ...
}
```

## Metrics

LocalAI exposes Prometheus metrics at `/metrics` when enabled via `--metrics` flag or `LOCALAI_METRICS=true`.

### Available Metrics

- `localai_inference_duration_seconds` — histogram of inference latency by model and backend
- `localai_requests_total` — counter of requests by endpoint and status
- `localai_model_load_duration_seconds` — time to load a model
- `localai_active_models` — gauge of currently loaded models
- `localai_tokens_generated_total` — total tokens generated

### Adding a New Metric

Define metrics in `pkg/metrics/metrics.go`:

```go
var myNewCounter = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "localai_my_feature_total",
        Help: "Total number of my feature operations.",
    },
    []string{"model", "status"},
)

func init() {
    prometheus.MustRegister(myNewCounter)
}
```

Increment it where appropriate:

```go
metrics.IncrementMyFeature(modelName, "success")
```

## Tracing

OpenTelemetry tracing is optional and enabled via `LOCALAI_OTEL_ENDPOINT`. Spans are created for:

- Model loading
- Backend gRPC calls
- HTTP request handling

### Adding a Span

```go
import "go.opentelemetry.io/otel"

tracer := otel.Tracer("localai/mypackage")
ctx, span := tracer.Start(ctx, "my-operation")
defer span.End()

span.SetAttributes(
    attribute.String("model", modelName),
    attribute.Int("max_tokens", req.MaxTokens),
)
```

## Health Checks

- `GET /healthz` — basic liveness probe, returns `200 OK` with `{"status": "ok"}`
- `GET /readyz` — readiness probe, returns `200` only when at least one backend is loaded

## Debugging Tips

- Set `LOG_LEVEL=debug` or `LOG_LEVEL=trace` for verbose output.
- Use `--log-file /tmp/localai.log` to persist logs across restarts.
- Backend subprocess logs are prefixed with the backend name, e.g., `[llama-cpp]`.
- Enable `DEBUG=true` to include request/response payloads in logs (avoid in production).
