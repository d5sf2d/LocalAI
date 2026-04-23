# Middleware and Security in LocalAI

This guide covers how to add middleware, authentication, rate limiting, and other security concerns to LocalAI API endpoints.

## Overview

LocalAI uses [Fiber](https://gofiber.io/) as its HTTP framework. Middleware is registered on the Fiber app or on specific route groups.

## Authentication Middleware

LocalAI supports API key authentication via the `Authorization` header or `x-api-key` header.

### How Auth Works

The auth middleware is defined in `pkg/http/middleware/auth.go`:

```go
package middleware

import (
    "github.com/gofiber/fiber/v2"
    "github.com/mudler/LocalAI/core/config"
)

// AuthMiddleware returns a Fiber middleware that validates API keys.
// If no API keys are configured, all requests are allowed through.
func AuthMiddleware(appConfig *config.ApplicationConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        if len(appConfig.ApiKeys) == 0 {
            return c.Next()
        }

        apiKey := c.Get("Authorization")
        if apiKey == "" {
            apiKey = c.Get("x-api-key")
        } else {
            // Strip "Bearer " prefix if present
            if len(apiKey) > 7 && apiKey[:7] == "Bearer " {
                apiKey = apiKey[7:]
            }
        }

        for _, key := range appConfig.ApiKeys {
            if key == apiKey {
                return c.Next()
            }
        }

        return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
            "error": fiber.Map{
                "message": "Invalid or missing API key",
                "type":    "invalid_request_error",
                "code":    "invalid_api_key",
            },
        })
    }
}
```

### Applying Auth to Routes

Apply the middleware to a route group, not individual routes:

```go
func RegisterMyRoutes(router fiber.Router, appConfig *config.ApplicationConfig) {
    // Apply auth to the entire group
    group := router.Group("/v1", middleware.AuthMiddleware(appConfig))
    group.Post("/my-endpoint", MyHandler)
}
```

## CORS Middleware

CORS is configured globally in the app setup. If you need custom CORS for a specific route group:

```go
import "github.com/gofiber/fiber/v2/middleware/cors"

group := router.Group("/v1")
group.Use(cors.New(cors.Config{
    AllowOrigins: "*",
    AllowHeaders: "Origin, Content-Type, Accept, Authorization, x-api-key",
    AllowMethods: "GET, POST, PUT, DELETE, OPTIONS",
}))
```

## Request Logging Middleware

Use the structured logger (see `observability-and-logging.md`) inside a middleware:

```go
func RequestLogger() fiber.Handler {
    return func(c *fiber.Ctx) error {
        start := time.Now()
        err := c.Next()
        log.Info().\n            Str("method", c.Method()).\n            Str("path", c.Path()).\n            Int("status", c.Response().StatusCode()).\n            Dur("latency", time.Since(start)).\n            Msg("request completed")
        return err
    }
}
```

## Rate Limiting

For endpoints that may be computationally expensive, use Fiber's built-in limiter:

```go
import "github.com/gofiber/fiber/v2/middleware/limiter"

group.Use(limiter.New(limiter.Config{
    Max:        60,             // 60 requests
    Expiration: 1 * time.Minute,
    KeyGenerator: func(c *fiber.Ctx) string {
        return c.IP() // Rate limit per IP
    },
    LimitReached: func(c *fiber.Ctx) error {
        return c.Status(fiber.StatusTooManyRequests).JSON(fiber.Map{
            "error": "rate limit exceeded",
        })
    },
}))
```

## Input Validation

Always validate and sanitize inputs before passing them to backends:

```go
func validateRequest(req *MyRequest) error {
    if req.Model == "" {
        return fmt.Errorf("model is required")
    }
    if req.MaxTokens < 0 {
        return fmt.Errorf("max_tokens must be non-negative")
    }
    // Prevent path traversal in model names
    if strings.Contains(req.Model, "..") || strings.Contains(req.Model, "/") {
        return fmt.Errorf("invalid model name")
    }
    return nil
}
```

## Security Headers

Add security headers globally or per-group:

```go
app.Use(func(c *fiber.Ctx) error {
    c.Set("X-Content-Type-Options", "nosniff")
    c.Set("X-Frame-Options", "DENY")
    c.Set("X-XSS-Protection", "1; mode=block")
    return c.Next()
})
```

## Middleware Order

Middleware is executed in the order it is registered. Recommended order:

1. Request logging
2. CORS
3. Security headers
4. Authentication
5. Rate limiting
6. Route handlers

## Testing Middleware

Use Fiber's `test` utility:

```go
func TestAuthMiddleware(t *testing.T) {
    app := fiber.New()
    cfg := &config.ApplicationConfig{ApiKeys: []string{"secret"}}
    app.Use(AuthMiddleware(cfg))
    app.Get("/test", func(c *fiber.Ctx) error {
        return c.SendString("ok")
    })

    // Test without key
    req := httptest.NewRequest("GET", "/test", nil)
    resp, _ := app.Test(req)
    assert.Equal(t, 401, resp.StatusCode)

    // Test with valid key
    req.Header.Set("Authorization", "Bearer secret")
    resp, _ = app.Test(req)
    assert.Equal(t, 200, resp.StatusCode)
}
```

## Related

- `api-endpoints-and-auth.md` — endpoint registration patterns
- `error-handling.md` — consistent error responses
- `observability-and-logging.md` — structured logging in middleware
