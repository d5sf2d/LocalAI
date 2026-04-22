# Model Configuration Guide

This guide explains how to configure models in LocalAI, including YAML configuration files, model parameters, and backend-specific settings.

## Overview

LocalAI uses YAML configuration files to define how models are loaded and served. Each model can have its own configuration file that specifies the backend, model path, and inference parameters.

## Configuration File Location

Model configuration files are stored in the models directory (default: `./models`). Each file should be named `<model-name>.yaml`.

## Basic Configuration Structure

```yaml
# Model display name
name: my-model

# Path to the model file (relative to models directory or absolute)
parameters:
  model: model.gguf

# Backend to use for inference
backend: llama-cpp

# Context size (number of tokens)
context_size: 4096

# Number of GPU layers to offload (-1 for all)
gpu_layers: 35

# Number of threads for CPU inference
threads: 4
```

## Backend Options

Available backends depend on your build configuration:

| Backend | Description | Model Format |
|---------|-------------|---------------|
| `llama-cpp` | llama.cpp based inference | GGUF |
| `whisper` | Speech-to-text | GGML/Whisper |
| `stablediffusion` | Image generation | Diffusion models |
| `bert-embeddings` | Text embeddings | GGML |
| `rwkv` | RWKV language models | GGML |
| `piper` | Text-to-speech | Piper voice models |

## Full Configuration Reference

```yaml
name: my-model

# Backend selection
backend: llama-cpp

# Model file parameters
parameters:
  model: llama-2-7b-chat.Q4_K_M.gguf
  # Sampling parameters (can be overridden per-request)
  temperature: 0.7
  top_k: 40
  top_p: 0.95
  max_tokens: 512
  repeat_penalty: 1.1

# Hardware configuration
context_size: 4096
gpu_layers: 35
threads: 4
batch_size: 512

# Memory mapping
mmap: true
mmlock: false

# Prompt template (for chat models)
template:
  chat: |
    {{.Input}}
  chat_message: |
    {{if eq .RoleName "user"}}[INST] {{.Content}} [/INST]{{end}}
    {{if eq .RoleName "assistant"}}{{.Content}}{{end}}
    {{if eq .RoleName "system"}}<<SYS>>\n{{.Content}}\n<</SYS>>\n\n{{end}}

# Stop words — model will stop generating at these tokens
stop_words:
  - "[INST]"
  - "</s>"

# Feature flags
embeddings: false
imagegen: false

# Roles mapping (optional)
roles:
  user: "USER"
  assistant: "ASSISTANT"
  system: "SYSTEM"
```

## Chat Template Variables

Templates use Go's `text/template` syntax. Available variables:

- `{{.Input}}` — Full conversation history formatted as a string
- `{{.RoleName}}` — Current message role (`user`, `assistant`, `system`)
- `{{.Content}}` — Current message content
- `{{.MessageIndex}}` — Index of current message in conversation

## Environment Variable Overrides

Some configuration values can be overridden via environment variables:

```bash
# Override models directory
LOCALAI_MODELS_PATH=/path/to/models

# Override default context size
LOCALAI_CONTEXT_SIZE=8192

# Override default number of threads
LOCALAI_THREADS=8

# Override default GPU layers
LOCALAI_GPU_LAYERS=40
```

## Validation

LocalAI validates configuration files on startup. Common issues:

- **Missing model file**: Ensure the model path is correct and the file exists
- **Invalid backend**: Check that the backend is compiled into your LocalAI build
- **Template syntax errors**: Validate Go template syntax carefully

## Example: OpenAI-Compatible Chat Model

```yaml
name: gpt-3.5-turbo
backend: llama-cpp
parameters:
  model: mistral-7b-instruct-v0.2.Q5_K_M.gguf
  temperature: 0.7
  top_p: 0.95
context_size: 32768
gpu_layers: 35
template:
  chat_message: |
    {{if eq .RoleName "system"}}<s>[INST] {{.Content}}\n{{end}}
    {{if eq .RoleName "user"}}{{if .MessageIndex}}[INST] {{end}}{{.Content}} [/INST]{{end}}
    {{if eq .RoleName "assistant"}}{{.Content}}</s>{{end}}
stop_words:
  - "[INST]"
  - "</s>"
```

## See Also

- [Adding Backends](.agents/adding-backends.md)
- [Gallery Models](.agents/adding-gallery-models.md)
- [API Endpoints](.agents/api-endpoints-and-auth.md)
