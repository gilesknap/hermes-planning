# Hermes Agent Setup & Management

This VPS is used to host a [Hermes Agent](https://github.com/NousResearch/hermes-agent) instance.
The Claude MAX 100 subscription powering this Claude Code session will also provide API tokens for the agent.

## Quick Reference

### What Is Hermes Agent?
An open-source, self-improving AI agent framework by Nous Research (MIT license, Python 3.11+).
Key features: autonomous skill creation, persistent cross-session memory, 70+ built-in tools,
15+ messaging platform gateways, multi-agent delegation, built-in cron scheduler.

- **Repo:** https://github.com/NousResearch/hermes-agent
- **Latest release (as of 2026-05-25):** v0.14.0
- **Default model:** `anthropic/claude-opus-4.6` (configurable)

### System Requirements
- Python 3.11+
- Node.js 20+ (optional: browser tools, WhatsApp adapter, TUI)
- Git with `--recurse-submodules` and `git-lfs`
- `uv` package manager

### Installation

**One-liner:**
```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

**Dev install:**
```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
uv venv venv --python 3.11
export VIRTUAL_ENV="$(pwd)/venv"
uv pip install -e ".[all,dev]"
npm install  # optional, for TUI/browser tools
```

**Docker:**
```bash
# docker-compose.yml provides two services: gateway (agent) + dashboard (web UI)
# Volume: ~/.hermes:/opt/data, network mode: host
docker compose up -d
```

### Configuration

- **Main config:** `~/.hermes/config.yaml` (from `cli-config.yaml.example`)
- **Secrets/API keys:** `~/.hermes/.env` (from `.env.example`, ~350+ vars documented)
- **Post-install:** `hermes setup` (interactive wizard) or manual config
- **Health check:** `hermes doctor`
- **Directory structure:** `~/.hermes/{cron,sessions,logs,memories,skills}/`

### Key Environment Variables
| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Direct Claude access |
| `OPENROUTER_API_KEY` | OpenRouter (200+ models) |
| `OPENAI_API_KEY` | OpenAI models |
| `GOOGLE_GEMINI_API_KEY` | Google Gemini |
| `FIRECRAWL_API_KEY` | Firecrawl web scraping/crawling |
| `EXA_API_KEY` | Exa AI-powered web search |

### External Tool Services

**Firecrawl** (https://firecrawl.dev) -- Web scraping API that converts pages to
clean markdown. Used by Hermes' web_extract toolset for high-quality page content
extraction (handles JS-rendered pages, removes boilerplate). Free tier: 500 credits/month.
Sign up at https://firecrawl.dev, get API key from dashboard.

**Exa** (https://exa.ai) -- AI-native search API. Returns semantically relevant
results with optional full-page content. Used by Hermes' web_search toolset as an
alternative/complement to traditional search. Free tier: 1000 searches/month.
Sign up at https://dashboard.exa.ai, get API key from settings.

### Supported LLM Providers (29 total)
Anthropic, OpenAI, OpenRouter, Google Gemini, DeepSeek, xAI, Nous, NVIDIA NIM,
Hugging Face, Ollama, Azure Foundry, AWS Bedrock, Alibaba, Qwen, Xiaomi, Kimi,
MiniMax, NovitaAI, Arcee, StepFun, Copilot, and more. Any OpenAI-compatible
endpoint via "Custom" provider.

### Gateway Platforms
Telegram, Discord, Slack, WhatsApp, Signal, Email, SMS, Matrix, DingTalk,
Feishu/Lark, WeCom, Weixin, Microsoft Teams, Home Assistant, QQ Bot,
BlueBubbles (iMessage), webhook, API server.

### Architecture Overview
- **Core engine** (`run_agent.py`): conversation loop, LLM calls via OpenAI-compatible API, SQLite session persistence with FTS5
- **CLI** (`cli.py`): Rich TUI with slash-commands, streaming, themes
- **Alt TUI** (`ui-tui/`): Ink/React terminal UI over JSON-RPC stdio bridge
- **Tools** (`tools/`): 70+ self-registering tools in ~30 toolset groups
- **Gateway** (`gateway/`): multi-platform messaging adapters
- **Skills** (`skills/`, `optional-skills/`): SKILL.md-based, community hub at agentskills.io
- **Plugins** (`plugins/`): memory providers (honcho, mem0, etc.) and model providers
- **Delegation**: subagent spawning (single or parallel batch), kanban collaboration

### Tool Categories
Terminal execution (7 backends: local/Docker/SSH/Singularity/Modal/Daytona/PTY),
file ops, browser automation (Playwright/Camofox/CDP), web search/extract,
code execution (sandboxed), vision, image/video gen, TTS, memory, skills mgmt,
cron jobs, delegation, kanban, MCP integration, computer-use, Home Assistant, Discord.

### Security Features
- Dangerous command detection with approval flow
- Cron prompt injection scanner
- Write deny list with symlink bypass prevention
- Skills guard + Tirith security scanning
- Tool loop guardrails (soft warnings + hard stops)
- Container hardening (dropped caps, no privesc, PID limits)
- API key stripping from child environments

### Useful Commands
```bash
hermes                # start interactive session
hermes setup          # configuration wizard
hermes doctor         # verify installation
hermes model          # switch model at runtime
hermes --tui          # launch Ink-based TUI
hermes skills install # install optional/community skills
```
