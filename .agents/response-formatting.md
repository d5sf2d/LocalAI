# Response Formatting in LocalAI

This guide covers how LocalAI formats and serializes responses for API consumers,
including OpenAI-compatible response structures, token usage reporting, and
consistent error/success envelope patterns.

## Overview

All API responses in LocalAI follow OpenAI-compatible JSON structures where possible.
Response formatting helpers live in `pkg/api/response` and are shared across endpoints.

## Core Response Types

```go
// pkg/api/response/types.go
package response

import "time"

// Usage tracks token consumption for a request.
type Usage struct {
	PromptTokens     int `json:"prompt_tokens"`
	CompletionTokens int `json:"completion_tokens"`
	TotalTokens      int `json:"total_tokens"`
}

// Choice represents a single completion choice.
type Choice struct {
	Index        int     `json:"index"`
	Message      Message `json:"message,omitempty"`
	Delta        Message `json:"delta,omitempty"`
	FinishReason string  `json:"finish_reason"`
}

// Message is a chat message with role and content.
type Message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

// ChatCompletion is the top-level chat response envelope.
type ChatCompletion struct {
	ID      string   `json:"id"`
	Object  string   `json:"object"`
	Created int64    `json:"created"`
	Model   string   `json:"model"`
	Choices []Choice `json:"choices"`
	Usage   Usage    `json:"usage"`
}

// NewChatCompletion constructs a ChatCompletion with sensible defaults.
func NewChatCompletion(id, model, content string, usage Usage) ChatCompletion {
	return ChatCompletion{
		ID:      id,
		Object:  "chat.completion",
		Created: time.Now().Unix(),
		Model:   model,
		Choices: []Choice{
			{
				Index:        0,
				Message:      Message{Role: "assistant", Content: content},
				FinishReason: "stop",
			},
		},
		Usage: usage,
	}
}
```

## Streaming Chunks

For streaming responses, use `ChatCompletionChunk`:

```go
// ChatCompletionChunk is a single SSE chunk for streaming.
type ChatCompletionChunk struct {
	ID      string   `json:"id"`
	Object  string   `json:"object"`
	Created int64    `json:"created"`
	Model   string   `json:"model"`
	Choices []Choice `json:"choices"`
}

func NewChunk(id, model, delta string, finishReason string) ChatCompletionChunk {
	return ChatCompletionChunk{
		ID:      id,
		Object:  "chat.completion.chunk",
		Created: time.Now().Unix(),
		Model:   model,
		Choices: []Choice{
			{
				Index:        0,
				Delta:        Message{Role: "assistant", Content: delta},
				FinishReason: finishReason,
			},
		},
	}
}

// DoneChunk returns the terminal [DONE] sentinel for SSE streams.
func DoneChunk() string { return "data: [DONE]\n\n" }
```

## Embedding Response

```go
// EmbeddingData holds a single embedding vector.
type EmbeddingData struct {
	Object    string    `json:"object"`
	Embedding []float32 `json:"embedding"`
	Index     int       `json:"index"`
}

// EmbeddingResponse is the top-level embedding response envelope.
type EmbeddingResponse struct {
	Object string          `json:"object"`
	Data   []EmbeddingData `json:"data"`
	Model  string          `json:"model"`
	Usage  Usage           `json:"usage"`
}
```

## Formatting Helpers

### JSON serialization

Always use `c.JSON()` from Fiber — never manually marshal inside handlers:

```go
// Good
return c.JSON(response.NewChatCompletion(id, model, text, usage))

// Avoid
bytes, _ := json.Marshal(resp)
c.Send(bytes)
```

### Content-Type

Fiber sets `application/json` automatically with `c.JSON()`. For SSE streams set
it explicitly before writing:

```go
c.Set("Content-Type", "text/event-stream")
c.Set("Cache-Control", "no-cache")
c.Set("Connection", "keep-alive")
```

### Generating Response IDs

Use a short UUID prefix to stay compatible with OpenAI tooling:

```go
import "github.com/google/uuid"

func NewResponseID() string {
	return "chatcmpl-" + uuid.New().String()
}
```

## Error Envelope

See `.agents/error-handling.md` for the `errorResponse` helper. All error
responses must use:

```json
{
  "error": {
    "message": "human readable message",
    "type":    "invalid_request_error",
    "code":    400
  }
}
```

## Checklist

- [ ] Use `response.NewChatCompletion` / `NewChunk` instead of inline structs
- [ ] Always populate `Usage` even if values are 0
- [ ] Set `finish_reason` to `"stop"` for normal completions, `"length"` when
      truncated by `max_tokens`
- [ ] Streaming handlers must send `DoneChunk()` as the final SSE event
- [ ] Never expose internal model paths or backend names in response fields
