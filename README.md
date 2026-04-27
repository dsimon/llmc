# llmc

A shell script launcher for LLM coding assistants. Presents an interactive menu to select from local models (via LM Studio) or cloud models (Anthropic, OpenAI, DeepInfra, OpenRouter, and others), then launches your chosen CLI with the right environment configured — no manual environment variable juggling required.

## Supported backends

| Command | CLI |
|---------|-----|
| `llmc` or `llmc claude` | [Claude Code](https://claude.ai/code) (Anthropic) |
| `llmc codex` | [Codex CLI](https://github.com/openai/codex) (OpenAI) |
| `llmc opencode` | [opencode](https://opencode.ai) |
| `llmc aider` | [aider](https://aider.chat) |

## What it does

- **Local models**: manages the full LM Studio lifecycle — runs `lms daemon up` to start LM Studio in headless mode if it isn't already running, queries the API to list available LLMs with architecture, quantization, size, and context info, loads the selected model before launching the CLI, and runs `lms daemon down` on exit (only if llmc started it).
- **Cloud models**: reads `~/.llmc/config.json` (your credentials, never committed to git) to build the menu. For the `claude` backend with OpenAI-compatible providers, starts a local [LiteLLM](https://github.com/BerriAI/litellm) proxy that translates Claude Code's Anthropic-format requests into OpenAI chat/completions format.
- **Option 0**: launches the backend with no model override, using whatever the CLI defaults to.
- **Clean environment**: saves all relevant env vars before launch and restores them on exit — your shell is left exactly as it was.

## Installation

```bash
# Clone
git clone https://github.com/dsimon/llmc.git
cd llmc

# Make executable and put on PATH
chmod +x llmc
ln -s "$PWD/llmc" /usr/local/bin/llmc   # or wherever you keep local scripts
```

**Dependencies:**

| Dependency | Required for |
|------------|-------------|
| `bash` 3+ | macOS ships bash 3 |
| `python3` | LM Studio model parsing, config loading |
| `curl` | LM Studio API calls |
| `bc` | Human-readable model size formatting |
| `lms` | LM Studio daemon management and model load/unload (optional; from [LM Studio](https://lmstudio.ai)) |
| `litellm` | openai_compatible providers with `claude` backend |

Install LiteLLM:
```bash
uv tool install 'litellm[proxy]'
```

## Configuration

On first run, `~/.llmc/config.json` is created from a template. Edit it to add your API keys and model list.

```
~/.llmc/
└── config.json    # credentials + model list — gitignored, never leaves your machine
```

### Provider types

```jsonc
{
  "providers": {
    "anthropic": {
      "name": "Anthropic Pro",
      "type": "anthropic",       // sets ANTHROPIC_API_KEY; used directly by claude backend
      "api_key": ""              // leave empty to use ANTHROPIC_API_KEY from environment
    },
    "openai": {
      "name": "ChatGPT Plus",
      "type": "openai",          // sets OPENAI_API_KEY; used directly by codex/aider/opencode
      "api_key": ""
    },
    "deepinfra": {
      "name": "DeepInfra",
      "type": "openai_compatible", // routes through LiteLLM proxy for claude backend
      "api_key": "your-key",
      "base_url": "https://api.deepinfra.com/v1/openai"
    },
    "openrouter": {
      "name": "OpenRouter",
      "type": "openai_compatible",
      "api_key": "your-key",
      "base_url": "https://openrouter.ai/api/v1"
    }
  }
}
```

### Adding models

Models are listed per backend. Each entry references a provider and uses that provider's model ID format:

| Provider | Model ID format | Example |
|----------|----------------|---------|
| `anthropic` | Anthropic canonical | `claude-sonnet-4-6` |
| `openai` | OpenAI canonical | `gpt-4o`, `o4-mini` |
| `deepinfra` | HuggingFace-style `Org/Model` | `Qwen/Qwen2.5-Coder-32B-Instruct` |
| `openrouter` | `provider/model` with dots for versions | `anthropic/claude-opus-4.7`, `qwen/qwen3.5-35b-a3b` |

**OpenRouter note**: version numbers use dots, not dashes (`anthropic/claude-opus-4.7`, not `claude-opus-4-7`). Verify IDs at [openrouter.ai/models](https://openrouter.ai/models).

Model lists support `_comment` keys for documentation — these are ignored by the parser:

```json
"claude": {
  "_comment": ["Direct Anthropic → OpenRouter failover → DeepInfra open-source"],
  "list": [
    {"provider": "anthropic",  "model": "claude-sonnet-4-6",        "display": "Claude Sonnet 4.6"},
    {"provider": "openrouter", "model": "anthropic/claude-sonnet-4.6","display": "Claude Sonnet 4.6"},
    {"provider": "deepinfra",  "model": "Qwen/Qwen2.5-Coder-32B-Instruct", "display": "Qwen2.5-Coder-32B"}
  ]
}
```

## Usage

```
llmc [claude|codex|opencode|aider] [local-llm-url]
```

```
  0    Default model (no override)

Local LLMs (from LM Studio):
  1    Qwen2.5-Coder-32B   Qwen    Q4_K_M (4bit)   19.8 GB   32768
  ...

Cloud LLMs (Anthropic API):
  2    Claude Opus 4.7 [Anthropic Pro]          (claude-opus-4-7)
  3    Claude Sonnet 4.6 [Anthropic Pro]        (claude-sonnet-4-6)
  4    Claude Opus 4.7 [OpenRouter]             (anthropic/claude-opus-4.7)
  5    Qwen3.5-35B-A3B [DeepInfra]              (qwen/qwen3.5-35b-a3b)
  ...

  Q/E  Quit / Exit

Choose a model (0-5, Q/E to quit) [0]:
```

Press Enter to accept the default (option 0) and launch with no model override.

### Remote LM Studio

Pass a second argument to connect to LM Studio running on another machine:

```bash
llmc claude 192.168.1.50         # expands to http://192.168.1.50:1234
llmc claude http://nas:1234
```

## Maintenance

### Updating model lists

Edit `~/.llmc/config.json` directly. No script changes needed — the config is reloaded on every run.

**Finding model IDs:**
- Anthropic: [platform.claude.com/docs/about-claude/models](https://platform.claude.com/docs/about-claude/models/overview)
- OpenAI: [platform.openai.com/docs/models](https://platform.openai.com/docs/models)
- DeepInfra: [deepinfra.com/models](https://deepinfra.com/models)
- OpenRouter: [openrouter.ai/models](https://openrouter.ai/models)

### Adding a new cloud provider

1. Add a provider block to `~/.llmc/config.json`
2. Add model entries referencing it under each backend's `list`

For the `claude` backend, any `openai_compatible` provider works automatically via the LiteLLM proxy. For OpenRouter specifically, llmc uses LiteLLM's native `openrouter/` provider to avoid routing restrictions.

### Updating LiteLLM

```bash
uv tool upgrade litellm
```

The LiteLLM proxy log path is shown in the terminal when a proxy session starts — useful for diagnosing connection issues with external providers. The log is created as a unique temp file and cleaned up on exit.

### Adding a new backend CLI

The script has a straightforward pattern for each backend in three places: model display label, environment setup, and the final launch command. Search for `aider` as a reference — it's the most recently added backend.

---

## Changelog

### April 2026 — Security hardening

- **Eval injection fix**: LM Studio API response values (model keys, display names, etc.) are now single-quote-escaped before being passed to `eval`, preventing command injection via a malicious or attacker-controlled LM Studio server.
- **API key exposure fix**: the opencode provider `api_key` is now passed via environment variable instead of a Python `sys.argv` argument, preventing it from appearing in `ps aux` output.
- **LiteLLM log**: proxy log is now created as a unique temp file (`mktemp`) and cleaned up on exit, replacing the fixed world-readable `/tmp/llmc-litellm.log` path.

### April 2026 — LM Studio daemon lifecycle management

- llmc now starts LM Studio in headless mode (`lms daemon up`) if it isn't already running, waits for the API to become available, then shuts the daemon down on exit (`lms daemon down`). If LM Studio was already running before llmc launched, it is left running on exit.

### April 2026 — Config-driven cloud models + LiteLLM proxy

- **Config file**: cloud models and credentials moved to `~/.llmc/config.json` (gitignored). Supports Anthropic Pro, ChatGPT Plus, DeepInfra, OpenRouter, Google AI Studio, Gemini — and any OpenAI-compatible endpoint.
- **LiteLLM proxy**: for the `claude` backend with `openai_compatible` providers, llmc now starts a local LiteLLM proxy that translates Claude Code's Anthropic Messages API requests into OpenAI chat/completions format, then forwards to the provider.
- **Option 0 / default launch**: pressing Enter at the menu launches the backend with no model override.
- **OpenRouter fix**: uses LiteLLM's native `openrouter/` provider to avoid incorrect provider-routing restrictions.
- **Auth**: uses `ANTHROPIC_AUTH_TOKEN` (not `ANTHROPIC_API_KEY`) for the proxy dummy credential, eliminating the auth-conflict warning.
- **Commented config**: model lists in `config.json` support `_comment` keys alongside the `list` array.

### Earlier 2026 — Multi-backend expansion

- **aider** added as a 4th backend (`llmc aider`)
- **opencode** added as a 3rd backend with cloud and local LM Studio support
- **Remote LM Studio**: optional second argument sets a custom LM Studio URL for remote instances
- **Q/E quit**: replaced numbered quit option with Q or E input
- **lms unload fix**: `lms unload --all` instead of bare model key identifier
- LM Studio connection warning suppressed — "No local LLMs found" is sufficient

### Initial release — Claude Code + Codex launcher

- Interactive menu for local LM Studio models and cloud models
- **claude** backend: Claude Code with Anthropic API (Opus, Sonnet, Haiku)
- **codex** backend: OpenAI Codex CLI
- Environment save/restore on exit — original env is always cleanly restored
- Local model load/unload via `lms` with human-readable size, context, and quantization info
