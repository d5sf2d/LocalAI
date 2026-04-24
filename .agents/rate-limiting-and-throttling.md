# Rate Limiting and Throttling in LocalAI

This guide covers how to implement and configure rate limiting and throttling for LocalAI API endpoints.

## Overview

LocalAI supports per-client and global rate limiting to prevent abuse and ensure fair resource allocation across consumers. Rate limiting is implemented as Fiber middleware and can be configured per-route or globally.

## Core Concepts

### Rate Limiter Middleware

LocalAI uses `github.com/gofiber/fiber/v2/middleware/limiter` for HTTP-level rate limiting:

```go
package middleware

import (
    "time"

    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/limiter"
    "github.com/rs/zerolog/log"
)

// RateLimiterConfig holds configuration for the rate limiter.
type RateLimiterConfig struct {
    // Max number of requests per Expiration window.
    Max int
    // Expiration is the time window for the rate limit.
    Expiration time.Duration
    // KeyGenerator generates a unique key per client.
    // Defaults to client IP if nil.
    KeyGenerator func(*fiber.Ctx) string
}

// NewRateLimiter returns a Fiber middleware that enforces rate limits.
func NewRateLimiter(cfg RateLimiterConfig) fiber.Handler {
    if cfg.Max == 0 {
        cfg.Max = 100
    }
    if cfg.Expiration == 0 {
        cfg.Expiration = 1 * time.Minute
    }

    keyGen := cfg.KeyGenerator
    if keyGen == nil {
        keyGen = func(c *fiber.Ctx) string {
            return c.IP()
        }
    }

    return limiter.New(limiter.Config{
        Max:        cfg.Max,
        Expiration: cfg.Expiration,
        KeyGenerator: keyGen,
        LimitReached: func(c *fiber.Ctx) error {
            log.Warn().
                Str("ip", c.IP()).
                Str("path", c.Path()).
                Msg("rate limit reached")
            return c.Status(fiber.StatusTooManyRequests).JSON(fiber.Map{
                "error": "rate limit exceeded, please slow down",
            })
        },
    })
}
```

### Registering Rate Limiting

Apply rate limiting when registering routes:

```go
func RegisterMyRoutes(app *fiber.App, cfg RateLimiterConfig) {
    rl := NewRateLimiter(cfg)

    // Apply globally
    app.Use(rl)

    // Or apply per route group
    api := app.Group("/v1")
    api.Use(rl)
    api.Post("/chat/completions", ChatEndpoint)
}
```

## Per-API-Key Rate Limiting

When authentication is enabled, rate limit by API key instead of IP:

```go
func APIKeyRateLimiter(max int, expiration time.Duration) fiber.Handler {
    return NewRateLimiter(RateLimiterConfig{
        Max:        max,
        Expiration: expiration,
        KeyGenerator: func(c *fiber.Ctx) string {
            // Extract Bearer token set by AuthMiddleware
            key := c.Locals("api_key")
            if key != nil {
                return key.(string)
            }
            return c.IP()
        },
    })
}
```

## Concurrency / Throttling for Model Inference

Beyond HTTP rate limiting, LocalAI throttles concurrent inference requests per model to avoid OOM errors:

```go
// ModelThrottle limits concurrent requests to a single model backend.
type ModelThrottle struct {
    sem chan struct{}
}

func NewModelThrottle(maxConcurrent int) *ModelThrottle {
    return &ModelThrottle{sem: make(chan struct{}, maxConcurrent)}
}

// Acquire blocks until a slot is available or context is cancelled.
func (t *ModelThrottle) Acquire(ctx context.Context) error {
    select {
    case t.sem <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

// Release frees a slot.
func (t *ModelThrottle) Release() {
    <-t.sem
}
```

Usage inside a request handler:

```go
func (h *Handler) ChatEndpoint(c *fiber.Ctx) error {
    if err := h.throttle.Acquire(c.Context()); err != nil {
        return c.Status(fiber.StatusServiceUnavailable).JSON(fiber.Map{
            "error": "server busy, try again later",
        })
    }
    defer h.throttle.Release()
    // ... run inference
}
```

## Configuration via Application Config

Expose rate limiting options in the application config struct:

```go
type ApplicationConfig struct {
    // ...
    // RateLimitMax is the max requests per RateLimitWindow per client.
    RateLimitMax    int           `env:"LOCALAI_RATE_LIMIT_MAX"    default:"0"`
    // RateLimitWindow is the sliding window duration for rate limiting.
    RateLimitWindow time.Duration `env:"LOCALAI_RATE_LIMIT_WINDOW" default:"1m"`
    // MaxConcurrentInference is the max simultaneous inference calls.
    MaxConcurrentInference int    `env:"LOCALAI_MAX_CONCURRENT"    default:"1"`
}
```

When `RateLimitMax` is 0, rate limiting is disabled.

## Testing Rate Limiting

```go
func TestRateLimiter(t *testing.T) {
    app := fiber.New()
    app.Use(NewRateLimiter(RateLimiterConfig{Max: 3, Expiration: time.Minute}))
    app.Get("/ping", func(c *fiber.Ctx) error { return c.SendString("pong") })

    for i := 0; i < 3; i++ {
        resp, err := app.Test(httptest.NewRequest("GET", "/ping", nil))
        require.NoError(t, err)
        require.Equal(t, 200, resp.StatusCode)
    }

    // 4th request should be throttled
    resp, err := app.Test(httptest.NewRequest("GET", "/ping", nil))
    require.NoError(t, err)
    require.Equal(t, 429, resp.StatusCode)
}
```

## Best Practices

- Set `RateLimitMax` conservatively for public deployments.
- Use API-key-based limiting when auth is enabled for per-user fairness.
- Keep `MaxConcurrentInference` at 1 for GPU-bound models unless you have multiple GPUs.
- Log rate limit events at `WARN` level for observability (see `observability-and-logging.md`).
- Return `Retry-After` headers when possible to help well-behaved clients back off.
