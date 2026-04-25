# Request Validation in LocalAI

This guide covers how to validate incoming API requests, sanitize inputs, and return structured errors for invalid payloads.

## Overview

LocalAI uses a layered validation approach:
1. **Struct-level validation** via `go-playground/validator` tags
2. **Business-logic validation** in handler functions
3. **Middleware-level checks** for common constraints (auth, content-type, size limits)

## Validation Tags

Use `validate:` struct tags for declarative field-level rules:

```go
type ChatRequest struct {
    Model       string        `json:"model" validate:"required,min=1,max=256"`
    Messages    []ChatMessage `json:"messages" validate:"required,min=1,dive"`
    MaxTokens   int           `json:"max_tokens" validate:"omitempty,min=1,max=32768"`
    Temperature float32       `json:"temperature" validate:"omitempty,min=0,max=2"`
    TopP        float32       `json:"top_p" validate:"omitempty,min=0,max=1"`
    Stream      bool          `json:"stream"`
}

type ChatMessage struct {
    Role    string `json:"role" validate:"required,oneof=system user assistant tool"`
    Content string `json:"content" validate:"required"`
}
```

## Shared Validator Instance

Initialize a single validator instance and reuse it across handlers:

```go
package api

import (
    "sync"
    "github.com/go-playground/validator/v10"
)

var (
    validate     *validator.Validate
    validateOnce sync.Once
)

func GetValidator() *validator.Validate {
    validateOnce.Do(func() {
        validate = validator.New()
        // Register JSON field names so errors reference JSON keys, not Go field names
        validate.RegisterTagNameFunc(func(fld reflect.StructField) string {
            name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
            if name == "-" {
                return ""
            }
            return name
        })
    })
    return validate
}
```

## Validating a Request in a Handler

```go
func ChatEndpoint(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) gin.HandlerFunc {
    return func(c *gin.Context) {
        var req schema.ChatCompletionRequest
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusBadRequest, schema.ErrorResponse{
                Error: &schema.APIError{
                    Code:    http.StatusBadRequest,
                    Message: "invalid JSON: " + err.Error(),
                    Type:    "invalid_request_error",
                },
            })
            return
        }

        if err := GetValidator().Struct(req); err != nil {
            c.JSON(http.StatusUnprocessableEntity, schema.ErrorResponse{
                Error: &schema.APIError{
                    Code:    http.StatusUnprocessableEntity,
                    Message: formatValidationErrors(err),
                    Type:    "invalid_request_error",
                },
            })
            return
        }

        // Continue with business logic...
    }
}
```

## Formatting Validation Errors

Convert `validator.ValidationErrors` into a human-readable string:

```go
func formatValidationErrors(err error) string {
    var ve validator.ValidationErrors
    if !errors.As(err, &ve) {
        return err.Error()
    }
    msgs := make([]string, 0, len(ve))
    for _, fe := range ve {
        msgs = append(msgs, fmt.Sprintf("field '%s' failed '%s' validation", fe.Field(), fe.Tag()))
    }
    return strings.Join(msgs, "; ")
}
```

## Body Size Limiting

Apply a body-size middleware before validation to prevent large payload attacks:

```go
func BodySizeLimit(maxBytes int64) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxBytes)
        if err := c.Request.ParseForm(); err != nil {
            c.AbortWithStatusJSON(http.StatusRequestEntityTooLarge, schema.ErrorResponse{
                Error: &schema.APIError{
                    Code:    http.StatusRequestEntityTooLarge,
                    Message: "request body too large",
                    Type:    "invalid_request_error",
                },
            })
            return
        }
        c.Next()
    }
}
```

Register it globally or per-route group:

```go
router.Use(BodySizeLimit(10 * 1024 * 1024)) // 10 MB
```

## Custom Validators

Register domain-specific rules once on the validator:

```go
func registerCustomValidators(v *validator.Validate) {
    // Ensure model name contains no path traversal characters
    _ = v.RegisterValidation("safemodelname", func(fl validator.FieldLevel) bool {
        name := fl.Field().String()
        return !strings.ContainsAny(name, "/\\..")
    })
}
```

Then use the tag in structs:

```go
Model string `json:"model" validate:"required,safemodelname"`
```

## Testing Validation

```go
func TestChatRequestValidation(t *testing.T) {
    v := GetValidator()

    valid := schema.ChatCompletionRequest{
        Model:    "gpt-4",
        Messages: []schema.ChatMessage{{Role: "user", Content: "hello"}},
    }
    assert.NoError(t, v.Struct(valid))

    missing := schema.ChatCompletionRequest{Messages: []schema.ChatMessage{{Role: "user", Content: "hi"}}}
    assert.Error(t, v.Struct(missing), "model is required")

    badTemp := schema.ChatCompletionRequest{
        Model:       "gpt-4",
        Messages:    []schema.ChatMessage{{Role: "user", Content: "hi"}},
        Temperature: 5.0, // out of range
    }
    assert.Error(t, v.Struct(badTemp))
}
```

## Best Practices

- Always bind JSON with `ShouldBindJSON` (returns error) rather than `BindJSON` (aborts automatically with less context).
- Return `422 Unprocessable Entity` for semantic validation failures and `400 Bad Request` for malformed JSON.
- Keep validation logic out of business-logic layers; validate at the HTTP boundary.
- Use `dive` tag on slices to validate each element.
- Log validation failures at `DEBUG` level — they are usually client errors, not server errors.
