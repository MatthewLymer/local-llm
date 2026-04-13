# Local OpenVINO Model Server for Claude Code

Local LLM inference stack for Claude Code using an Intel Arc A770 16GB GPU.

## Architecture

```
Claude Code  →  LiteLLM (:4000)  →  OVMS (:8000)  →  Intel Arc A770
(Anthropic API)  (translates)     (OpenAI /v3 API)    (GPU inference)
```

Two services are needed because OVMS speaks OpenAI format but Claude Code requires the Anthropic Messages API. LiteLLM handles the translation.

## Quick Start

```bash
# 1. Find your render group GID on the Intel GPU machine
stat -c "%g" /dev/dri/render*

# 2. Start (first run downloads ~5GB model)
RENDER_GROUP_GID=<gid> docker compose up -d

# 3. Watch model download progress
docker compose logs -f ovms

# 4. Configure Claude Code
export ANTHROPIC_BASE_URL=http://localhost:4000
export ANTHROPIC_API_KEY=sk-local-dev
```

## Configuration

Override these with environment variables or a `.env` file.

| Variable | Default | Purpose |
|---|---|---|
| `RENDER_GROUP_GID` | `109` | GID of `/dev/dri/renderD*` for GPU access |
| `OVMS_SOURCE_MODEL` | `OpenVINO/Qwen3-8B-Instruct-int4-ov` | HuggingFace model ID ([browse](https://huggingface.co/OpenVINO)) |
| `OVMS_MODEL_NAME` | `qwen3-8b` | Name exposed via the API |
| `OVMS_CACHE_SIZE_GB` | `8` | KV cache VRAM budget in GB |
| `OVMS_PORT` | `11434` | Host port for direct OVMS access |
| `LITELLM_PORT` | `4000` | Host port for the Anthropic-compatible proxy |
| `LITELLM_API_KEY` | `sk-local-dev` | API key (must match `ANTHROPIC_API_KEY` in Claude Code) |
| `HF_TOKEN` | _(none)_ | HuggingFace token (only needed for gated models) |

## Changing Models

```bash
# Stop the stack
docker compose down

# Set the new model
export OVMS_SOURCE_MODEL=OpenVINO/Qwen3-14B-Instruct-int4-ov
export OVMS_MODEL_NAME=qwen3-14b
export OVMS_CACHE_SIZE_GB=6

# Restart (downloads automatically on first start)
docker compose up -d
```

Previously downloaded models remain in the Docker volume.

### Supported Models

OVMS works best with pre-converted OpenVINO IR models from [HuggingFace/OpenVINO](https://huggingface.co/OpenVINO).

GGUF support is limited to: Qwen2/2.5/3, Llama3, and DeepSeek-R1-Distill architectures. Gemma, Mistral, and others are **not** supported as GGUF.

For Claude Code, models with strong tool-calling are essential. The **Qwen3 family** is recommended.

## Context Size Tuning

Context capacity is governed by `OVMS_CACHE_SIZE_GB` — the KV cache VRAM budget. More cache means longer context windows and/or more concurrent sequences.

For the Intel Arc A770 (16GB VRAM):

| Model (int4) | Model Size | Recommended `OVMS_CACHE_SIZE_GB` |
|---|---|---|
| Qwen3-4B | ~2.5 GB | 10–12 |
| Qwen3-8B | ~5.0 GB | 8–10 |
| Qwen3-14B | ~8.0 GB | 5–7 |

The `u8` KV cache precision (configured by default) halves memory per token compared to FP16. Qwen3-8B supports up to 32,768 tokens natively.

To verify actual capacity after startup:

```bash
docker compose logs ovms | grep -i "cache\|capacity\|context"
```

## Useful Endpoints

```bash
# List installed models (OVMS direct)
curl http://localhost:11434/v3/models

# List models via LiteLLM
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer sk-local-dev"

# Health checks
curl http://localhost:11434/v2/health/ready
curl http://localhost:4000/health

# Test chat completion
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-local-dev" \
  -H "Content-Type: application/json" \
  -d '{"model": "local-model", "messages": [{"role": "user", "content": "Hello"}]}'
```

## Caveats

- The LiteLLM Anthropic-to-OpenAI translation may not perfectly handle all Claude Code features (complex tool use, streaming edge cases). Check `docker compose logs litellm` for translation errors.
- A local Qwen3-8B is significantly less capable than Claude Sonnet/Opus for coding tasks. Results will vary.
- First startup is slow (model download + GPU optimization). Subsequent starts use the cached model from the Docker volume.
