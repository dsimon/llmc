# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains the `llmc` shell script, which acts as an LLM launcher for Claude Code and OpenAI Codex. It provides an interactive menu to switch between cloud models and local models running via LM Studio.

## Usage

```
llmc [claude|codex]   # default: claude
```

- `llmc` or `llmc claude` — launch Anthropic's Claude Code
- `llmc codex` — launch OpenAI's Codex CLI

No manual model pre-loading is needed — the script handles it automatically.

### Local Models (LM Studio)

The script queries `http://localhost:1234/api/v1/models` to fetch all installed models from a running LM Studio instance. When a local model is selected, the script uses `lms unload` to unload any currently loaded models and `lms load` to load the chosen model before launching the CLI. It then sets the necessary environment variables for local usage.

### Cloud Models

- **claude**: Anthropic Claude 4 series (Opus, Sonnet, Haiku)
- **codex**: OpenAI models (o4-mini, o3, GPT-4.1, GPT-4o, GPT-4o Mini)

## Environment Variables

The script manages and temporarily overrides the following environment variables:

**Claude:**
- `ANTHROPIC_MODEL`: The selected model identifier.
- `ANTHROPIC_BASE_URL`: Set to `http://localhost:1234` when using local models.
- `ANTHROPIC_AUTH_TOKEN`: Set to `lmstudio` for local models.

**Codex:**
- `OPENAI_MODEL`: The selected model identifier.
- `OPENAI_BASE_URL`: Set to `http://localhost:1234/v1` when using local models.
- `OPENAI_API_KEY`: Set to `lmstudio` for local models (preserves existing key for cloud).

The script uses a backup mechanism to ensure the original environment is restored upon exit or cleanup.

## Configuration

Permissions for interacting with LM Studio (e.g., `WebFetch`, `Bash` calls to localhost) are configured in `.claude/settings.local.json`.
