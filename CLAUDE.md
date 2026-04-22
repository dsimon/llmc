# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains the `claude-code` shell script, which acts as an LLM launcher for Claude Code. It provides an interactive menu to switch between Anthropic cloud models and local models running via LM Studio.

## Usage

Run the `./claude-code` script to launch the interactive model selection menu. No manual model pre-loading is needed — the script handles it automatically.

### Local Models (LM Studio)

The script queries `http://localhost:1234/api/v1/models` to fetch all installed models from a running LM Studio instance. When a local model is selected, the script uses `lms unload` to unload any currently loaded models and `lms load` to load the chosen model before launching Claude Code. It then sets the necessary environment variables for local usage.

### Cloud Models

The script supports selecting various Anthropic cloud models, including Claude 3.5 and Claude 3 series.

## Environment Variables

The script manages and temporarily overrides the following `ANTHROPIC_*` environment variables:

- `ANTHROPIC_MODEL`: The selected model identifier.
- `ANTHROPIC_BASE_URL`: Set to `http://localhost:1234` when using local models.
- `ANTHROPIC_AUTH_TOKEN`: Set to `lmstudio` for local models.

The script uses a backup mechanism to ensure the original environment is restored upon exit or cleanup.

## Configuration

Permissions for interacting with LM Studio (e.g., `WebFetch`, `Bash` calls to localhost) are configured in `.claude/settings.local.json`.
