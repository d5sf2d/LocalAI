# Caching and Performance in LocalAI

This guide covers caching strategies, performance optimization patterns, and best practices used throughout LocalAI.

## Response Caching

LocalAI supports caching at multiple layers to reduce redundant model inference.

### In-Memory Cache

Use a simple TTL-based cache for short-lived results:

```go
package cache

import (
	"sync"
	"time"
)

// CacheEntry holds a cached value with its expiration time.
type CacheEntry struct {
	Value     interface{}
	ExpiresAt time.Time
}

// TTLCache is a thread-safe in-memory cache with TTL support.
type TTLCache struct {
	mu      sync.RWMutex
	entries map[string]CacheEntry
	ttl     time.Duration
}

// NewTTLCache creates a new TTLCache with the given TTL duration.
func NewTTLCache(ttl time.Duration) *TTLCache {
	c := &TTLCache{
		entries: make(map[string]CacheEntry),
		ttl:     ttl,
	}
	go c.evictLoop()
	return c
}

// Set stores a value in the cache under the given key.
func (c *TTLCache) Set(key string, value interface{}) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.entries[key] = CacheEntry{
		Value:     value,
		ExpiresAt: time.Now().Add(c.ttl),
	}
}

// Get retrieves a value from the cache. Returns (value, true) if found and not expired.
func (c *TTLCache) Get(key string) (interface{}, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	entry, ok := c.entries[key]
	if !ok || time.Now().After(entry.ExpiresAt) {
		return nil, false
	}
	return entry.Value, true
}

// evictLoop periodically removes expired entries.
func (c *TTLCache) evictLoop() {
	ticker := time.NewTicker(c.ttl / 2)
	defer ticker.Stop()
	for range ticker.C {
		c.mu.Lock()
		now := time.Now()
		for k, v := range c.entries {
			if now.After(v.ExpiresAt) {
				delete(c.entries, k)
			}
		}
		c.mu.Unlock()
	}
}
```

## Request Deduplication

Avoid duplicate in-flight requests for the same prompt/model combination using `singleflight`:

```go
import "golang.org/x/sync/singleflight"

var sfGroup singleflight.Group

// InferWithDedup runs inference, deduplicating concurrent identical requests.
func InferWithDedup(key string, fn func() (interface{}, error)) (interface{}, error) {
	v, err, _ := sfGroup.Do(key, fn)
	return v, err
}
```

## Cache Key Generation

Cache keys should be deterministic and capture all inputs that affect the output:

```go
import (
	"crypto/sha256"
	"encoding/json"
	"fmt"
)

// CacheKey generates a stable cache key from a request struct.
func CacheKey(req interface{}) (string, error) {
	b, err := json.Marshal(req)
	if err != nil {
		return "", fmt.Errorf("cache key serialization: %w", err)
	}
	hash := sha256.Sum256(b)
	return fmt.Sprintf("%x", hash), nil
}
```

## Model Loading Performance

### Lazy Loading

Models are loaded on first use and kept resident in memory. See `backends/` for the loader pattern.

- Use `sync.Once` to ensure a model is only loaded once per process.
- Hold a reference count; unload after a configurable idle timeout.

### Preloading

Set `preload: true` in the model config YAML to load models at startup:

```yaml
name: my-model
backend: llama
parameters:
  model: models/my-model.gguf
preload: true
```

## HTTP Response Compression

Enable gzip compression for non-streaming responses by adding the middleware:

```go
import "github.com/gofiber/fiber/v2/middleware/compress"

app.Use(compress.New(compress.Config{
	Level: compress.LevelBestSpeed,
}))
```

> **Note:** Do not apply compression to streaming (SSE) endpoints — it interferes with chunked delivery.

## Profiling

Enable the pprof endpoint in development builds:

```go
import "github.com/gofiber/contrib/fiberzerolog"

if os.Getenv("LOCALAI_PPROF") == "true" {
	app.Get("/debug/pprof/*", adaptor.HTTPHandler(http.DefaultServeMux))
}
```

Run a CPU profile:

```bash
go tool pprof http://localhost:8080/debug/pprof/profile?seconds=30
```

## Best Practices

| Concern | Recommendation |
|---|---|
| Repeated identical prompts | Use TTL cache keyed on SHA-256 of request |
| Concurrent duplicate requests | Use `singleflight` |
| Large model files | Memory-map via backend (llama.cpp does this by default) |
| Slow first request | Use `preload: true` in model config |
| JSON serialization hot path | Use `json-iterator` or `sonic` |
| Streaming endpoints | Skip caching and compression |

## Related Guides

- [Rate Limiting and Throttling](.agents/rate-limiting-and-throttling.md)
- [Streaming Responses](.agents/streaming-responses.md)
- [Model Configuration](.agents/model-configuration.md)
