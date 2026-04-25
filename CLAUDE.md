# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains the `llmc` shell script — an LLM launcher for Claude Code, OpenAI Codex, opencode, and aider. It presents an interactive menu to select from local models (via LM Studio) or cloud models, then launches the chosen CLI with the appropriate environment configured.

## Usage

```
llmc [claude|codex|opencode|aider] [local-llm-url]   # default: claude, http://localhost:1234
```

- `llmc` or `llmc claude` — launch Claude Code (Anthropic)
- `llmc codex` — launch OpenAI Codex CLI
- `llmc opencode` — launch opencode
- `llmc aider` — launch aider

Option `0` (default, press Enter) launches the backend with no model override.

## Configuration File

Cloud models and credentials live in **`~/.llmc/config.json`** — never committed to git.

The config has two sections:

### `providers`
Named credential blocks. Each has:
- `type`: `"anthropic"`, `"openai"`, or `"openai_compatible"`
- `api_key`: provider key (leave empty to use existing env var)
- `base_url`: endpoint (openai_compatible only)

### `models`
Per-backend ordered lists of selectable models. Each entry:
```json
{"provider": "<provider-key>", "model": "<model-id>", "display": "<menu label>"}
```

The menu shows `"<display> [<provider.name>]"` for each entry. Each backend's list is a `list` array nested in an object (to allow `_comment` keys alongside it). The config template generated on first run includes full `_comment` documentation explaining all conventions.

### Model ID conventions by provider
| Provider | Format | Example |
|----------|--------|---------|
| `anthropic` | Anthropic canonical IDs | `claude-opus-4-7` |
| `openai` | OpenAI canonical IDs | `o4-mini`, `gpt-4o` |
| `deepinfra` | HuggingFace-style `Org/Model` | `Qwen/Qwen2.5-Coder-32B-Instruct` |
| `openrouter` | `provider/model` with dots for versions | `anthropic/claude-opus-4.7`, `openai/gpt-4o` |

**OpenRouter note**: version numbers use dots, not dashes (`claude-opus-4.7` not `claude-opus-4-7`). Verify current IDs at [openrouter.ai/models](https://openrouter.ai/models).

## Model Selection

### Local Models (LM Studio)
The script queries `http://localhost:1234/api/v1/models` and lists available LLMs with architecture, quantization, size, context length, and loaded status. On selection it runs `lms unload --all` then `lms load <key>` before launching the CLI.

### Cloud Models
Loaded from `~/.llmc/config.json`. Current default set per backend:
- **claude/openrouter**: Direct Anthropic API → OpenRouter Anthropic → DeepInfra Qwen → OpenRouter Qwen
- **codex/openrouter**: Direct OpenAI API → OpenRouter OpenAI → DeepInfra Qwen → OpenRouter Qwen
- **opencode/aider**: Same pattern across all providers

## How Cloud Model Routing Works

### `anthropic` providers (claude backend)
Sets `ANTHROPIC_API_KEY` and runs `claude --model <model-id>` directly.

### `openai` providers (codex/aider/opencode backends)
Sets `OPENAI_API_KEY` and runs the CLI directly. No base URL override.

### `openai_compatible` providers — claude backend
Claude Code always sends Anthropic-format requests (`POST /v1/messages`) regardless of model name. External providers only speak OpenAI chat/completions format. llmc bridges this by:

1. Starting a **LiteLLM proxy** (`uv tool install 'litellm[proxy]'` required) on a random localhost port
2. Setting `ANTHROPIC_BASE_URL=http://127.0.0.1:<port>` so Claude Code routes to the proxy
3. Setting `ANTHROPIC_AUTH_TOKEN=llmc-proxy` (dummy) to override the claude.ai session token without triggering the API-key conflict warning
4. LiteLLM translates Anthropic→OpenAI and forwards to the provider

LiteLLM is started with `LITELLM_USE_CHAT_COMPLETIONS_URL_FOR_ANTHROPIC_MESSAGES=true` and `drop_params: true` (strips unsupported params like `reasoning_effort`).

### `openai_compatible` providers — codex/aider backends
Sets `OPENAI_BASE_URL` and `OPENAI_API_KEY`. For aider the model is prefixed with `openai/` to select OpenAI-compatible mode.

### `openai_compatible` providers — opencode backend
Generates a temporary opencode JSON config file with a custom provider block and sets `OPENCODE_CONFIG` to point to it.

## Environment Variables Managed

The script saves and restores all of these on exit:

| Variable | Used by |
|----------|---------|
| `ANTHROPIC_MODEL` | claude (local models) |
| `ANTHROPIC_BASE_URL` | claude (local + openai_compatible cloud) |
| `ANTHROPIC_AUTH_TOKEN` | claude (local models + openai_compatible cloud proxy) |
| `ANTHROPIC_API_KEY` | claude (anthropic cloud) |
| `OPENAI_MODEL` | codex |
| `OPENAI_BASE_URL` | codex/aider (local + openai_compatible cloud) |
| `OPENAI_API_KEY` | codex/aider (openai + openai_compatible cloud) |
| `OPENCODE_CONFIG` | opencode (openai_compatible cloud) |

## Dependencies

| Dependency | Required for |
|------------|-------------|
| `python3` | LM Studio model parsing; LiteLLM config generation |
| `curl` | LM Studio model API fetch |
| `bc` | Human-readable model size formatting |
| `lms` | LM Studio model load/unload (optional; skipped if absent) |
| `litellm` | openai_compatible providers with claude backend |

Install LiteLLM: `uv tool install 'litellm[proxy]'`

## Files

| Path | Purpose |
|------|---------|
| `llmc` | Main script |
| `~/.llmc/config.json` | Cloud model config and credentials (gitignored) |
| `.claude/settings.local.json` | Claude Code permissions (localhost WebFetch, etc.) |
| `/tmp/llmc-litellm.log` | LiteLLM proxy log (written each session) |
| `/tmp/llmc-litellm.*.yaml` | Ephemeral LiteLLM config (cleaned up on exit) |
