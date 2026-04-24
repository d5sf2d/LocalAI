# Streaming Responses in LocalAI

This guide covers how to implement and handle streaming responses (Server-Sent Events / SSE) in LocalAI backends and API endpoints.

## Overview

LocalAI supports streaming for chat completions and other generation endpoints, following the OpenAI streaming protocol. Responses are sent as `text/event-stream` with `data: <json>\n\n` framing, terminated by `data: [DONE]\n\n`.

## Key Interfaces

### TokenCallback

Backends that support streaming implement a token callback pattern:

```go
// TokenCallback is called by the backend for each generated token.
// Returning false signals the backend to stop generation.
type TokenCallback func(token string, usage TokenUsage) bool
```

### StreamWriter

The HTTP layer uses `StreamWriter` to flush SSE chunks to the client:

```go
func StreamWriter(c *fiber.Ctx, channel <-chan schema.OpenAIResponse, done <-chan struct{}) error {
    c.Set("Content-Type", "text/event-stream")
    c.Set("Cache-Control", "no-cache")
    c.Set("Connection", "keep-alive")
    c.Set("Transfer-Encoding", "chunked")

    c.Context().SetBodyStreamWriter(fasthttp.StreamWriter(func(w *bufio.Writer) {
        for {
            select {
            case <-done:
                fmt.Fprintf(w, "data: [DONE]\\n\\n")
                w.Flush()
                return
            case response, ok := <-channel:
                if !ok {
                    fmt.Fprintf(w, "data: [DONE]\\n\\n")
                    w.Flush()
                    return
                }
                data, err := json.Marshal(response)
                if err != nil {
                    log.Error().Err(err).Msg("failed to marshal stream response")
                    continue
                }
                fmt.Fprintf(w, "data: %s\\n\\n", data)
                w.Flush()
            }
        }
    }))
    return nil
}
```

## Implementing Streaming in an Endpoint

### 1. Detect Stream Mode

```go
func ChatEndpoint(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        var req schema.OpenAIRequest
        if err := c.BodyParser(&req); err != nil {
            return fiber.ErrBadRequest
        }

        if req.Stream {
            return handleStreamingChat(c, req, cl, ml, appConfig)
        }
        return handleBlockingChat(c, req, cl, ml, appConfig)
    }
}
```

### 2. Set Up the Channel Pipeline

```go
func handleStreamingChat(
    c *fiber.Ctx,
    req schema.OpenAIRequest,
    cl *config.BackendConfigLoader,
    ml *model.ModelLoader,
    appConfig *config.ApplicationConfig,
) error {
    responses := make(chan schema.OpenAIResponse, 16)
    done := make(chan struct{})

    go func() {
        defer close(responses)
        defer close(done)

        cb := func(token string, usage TokenUsage) bool {
            // Check if client disconnected
            select {
            case <-c.Context().Done():
                return false
            default:
            }

            responses <- schema.OpenAIResponse{
                ID:      req.ID,
                Created: time.Now().Unix(),
                Model:   req.Model,
                Object:  "chat.completion.chunk",
                Choices: []schema.Choice{
                    {
                        Delta: &schema.Message{Content: token},
                        Index: 0,
                    },
                },
            }
            return true
        }

        if err := runInference(req, cb, cl, ml, appConfig); err != nil {
            log.Error().Err(err).Str("model", req.Model).Msg("streaming inference error")
        }
    }()

    return StreamWriter(c, responses, done)
}
```

## Client-Side Cancellation

LocalAI respects client disconnects during streaming. The `TokenCallback` checks `c.Context().Done()` before sending each token. When the client closes the connection, `fiber.Ctx.Context().Done()` is closed and the callback returns `false`, signalling the backend to halt generation.

## Partial Aggregation for Usage Stats

When `stream_options.include_usage` is set in the request, append a final chunk before `[DONE]`:

```go
if req.StreamOptions != nil && req.StreamOptions.IncludeUsage {
    responses <- schema.OpenAIResponse{
        // ... fields ...
        Usage: &schema.OpenAIUsage{
            PromptTokens:     usage.Prompt,
            CompletionTokens: usage.Completion,
            TotalTokens:      usage.Prompt + usage.Completion,
        },
    }
}
```

## Testing Streaming Endpoints

Use `net/http/httptest` with a pipe to consume SSE events:

```go
func TestStreamingResponse(t *testing.T) {
    app := fiber.New()
    app.Post("/v1/chat/completions", ChatEndpoint(cl, ml, appConfig))

    req := httptest.NewRequest(http.MethodPost, "/v1/chat/completions", body)
    req.Header.Set("Content-Type", "application/json")

    resp, err := app.Test(req, 30000)
    require.NoError(t, err)
    require.Equal(t, http.StatusOK, resp.StatusCode)
    require.Equal(t, "text/event-stream", resp.Header.Get("Content-Type"))

    scanner := bufio.NewScanner(resp.Body)
    var chunks []string
    for scanner.Scan() {
        line := scanner.Text()
        if strings.HasPrefix(line, "data: ") && line != "data: [DONE]" {
            chunks = append(chunks, strings.TrimPrefix(line, "data: "))
        }
    }
    require.NotEmpty(t, chunks)
}
```

## Common Pitfalls

- **Buffering**: Always call `w.Flush()` after each SSE event; Fiber/fasthttp may buffer otherwise.
- **Channel size**: Use a small buffer (8–32) to avoid blocking the inference goroutine on slow clients.
- **Error propagation**: Log errors inside the goroutine; do not return them via the channel (use a separate error channel if needed).
- **Goroutine leaks**: Ensure the inference goroutine exits when the client disconnects or generation completes.
