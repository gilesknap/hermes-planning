# Hermes PKM Agent — Specification

> **Executive summary.** The Hermes PKM Agent is a single-user "second brain" that lets its owner *throw content at it and forget*, then *ask for it back in plain language* — from Slack on a phone, or from Claude Code / claude.ai at a desk. It runs unattended on an isolated VPS and treats an **Obsidian vault of markdown files as the one canonical store**, organised as a hybrid of Andrej Karpathy's three-layer "LLM Wiki" (immutable raw sources → curated cross-linked pages → schema) and PKM life-admin domains (TODOs, a media-to-consume backlog, personal memories) represented as typed wiki pages surfaced through Obsidian Bases dashboards. Hindsight is a **rebuildable semantic index over the vault**, never the store; the Hermes SQLite session DB (`state.db`) is **transient working memory only**. The vault lives in its own dedicated private GitHub repo, two-way synced (Obsidian Git plugin on the workstation, pull-before-write / commit-after-ingest on the agent). Retrieval always points the user back to the real note via clickable `obsidian://` deep links. This document fully specifies the system — absorbing the prior `hermes-planning/CLAUDE.md` planning notes and the as-built `hermes-agent` deployment — and includes the migration plan from today's `documents/` proto-vault to the new vault.

## Table of Contents

1. [Overview and Guiding Principles](#1-overview-and-guiding-principles)
2. [User Profile and Constraints](#2-user-profile-and-constraints)
3. [System Architecture](#3-system-architecture)
4. [The Hermes Platform](#4-the-hermes-platform)
5. [As-Built Today: Inventory and What Must Change](#5-as-built-today-inventory-and-what-must-change)
6. [The PKM Vault (Canonical Store)](#6-the-pkm-vault-canonical-store)
7. [Capture / Ingestion](#7-capture--ingestion)
8. [Retrieval](#8-retrieval)
9. [Semantic Index (Hindsight)](#9-semantic-index-hindsight)
10. [Life-Admin](#10-life-admin)
11. [Proactive Summaries](#11-proactive-summaries)
12. [Sync, Repos and Backup](#12-sync-repos-and-backup)
13. [Maintenance](#13-maintenance)
14. [Security and Privacy](#14-security-and-privacy)
15. [Setup and Operations Runbook](#15-setup-and-operations-runbook)
16. [Migration Plan](#16-migration-plan)
17. [Non-Goals and Open Questions](#17-non-goals-and-open-questions)
18. [Appendix](#18-appendix)

---

## 1. Overview and Guiding Principles

The Hermes PKM Agent is a **second brain**: a single-user knowledge system that captures whatever its owner sends it, files it durably, and gives it back conversationally. It runs unattended on an isolated VPS. The everyday surface is a Slack DM; the same capabilities are exposed over MCP for use inside Claude Code / claude.ai. The owner never files or tags anything by hand — the agent decides type, layer, filename, tags, and links.

This section establishes the frame; later sections own the detailed schemas, the Hindsight tuning, the sync protocol, and the migration plan.

### Guiding principles

| # | Principle | What it means in practice |
|---|-----------|---------------------------|
| P1 | **Files are canonical.** | The vault (plain markdown + binary assets in `raw/assets/`) is the source of truth. Hindsight is a rebuildable index; `state.db` is transient working memory. If the index and the files disagree, the files win and the index is rebuilt. |
| P2 | **Knowledge compounds.** | Every capture lands as a durable, cross-linked page that future captures and queries can link to. The value is in the accreting graph, not in any single note or in chat history. |
| P3 | **Always human-browsable.** | Anything the agent writes must be readable and editable directly in Obsidian without the agent present. No opaque blobs, no base64 in notes, no agent-only formats. |
| P4 | **Semantic search is a layer, not the store.** | Structural navigation (folders, wikilinks, `index.md`, Bases dashboards) is primary; Hindsight semantic recall is layered on top to find pages by meaning. Losing the index loses nothing permanent. |
| P5 | **Throw-it-and-forget capture.** | The user sends a URL, file, photo, thought, or TODO with zero ceremony. The agent decides type, layer, filename, tags, and links. The user is never asked to file or tag. |
| P6 | **Auto-categorisation with a stable taxonomy.** | Captures are routed into the hybrid taxonomy (llm-wiki knowledge layers + life-admin `type` pages surfaced as Bases views) deterministically enough that the same input always lands in the same place — so collisions stay rare and two-way git sync stays clean. |
| P7 | **Retrieval points back to the original.** | Answers are not the end state. Every retrieval offers a way to **open the actual note or attachment in Obsidian** on the user's device (see §8), because the vault — not the chat — is where the user does real work. |
| P8 | **Cheap, autonomous, separated.** | High autonomy (`approvals.mode=auto`) on an isolated box; inference on a pay-as-you-go Anthropic key kept strictly apart from the user's Claude Max subscription; embeddings on Gemini's free tier. |

This is the architectural retirement of the old "the DB is the truth" model that the current `SOUL.md` encodes. Under this spec, the vault is the truth; the session DB and the Hindsight index are disposable projections of it.

---

## 2. User Profile and Constraints

**Single user — this is not a multi-tenant product.**

| Attribute | Value |
|-----------|-------|
| Name | Giles Knap |
| Email | gilesknap@gmail.com |
| Timezone | Europe/London (GMT/BST) — all schedules and relative dates resolve here |
| Slack user ID | `U0B5ZP301H8` (username `giles.knap`) |
| Primary Slack DM | channel `D0B61LKA3NV` (the agent's DM with the user; summary delivery target) |
| Interactive Claude | Claude Max subscription via claude.ai / Claude Code — **personal use only** |
| Agent inference | **Separate** pay-as-you-go Anthropic API key (`ANTHROPIC_API_KEY`) |
| Embeddings | Google Gemini free tier |
| Hosting | Isolated, single-tenant VPS |

### Hard constraints (these shape every later decision)

- **Two billing streams must never cross.** The Claude Max OAuth credential is for interactive Claude only. Hermes authenticates **exclusively** via the pay-as-you-go `ANTHROPIC_API_KEY`. The Claude Max OAuth token must **never** be wired into the agent — doing so risks an Anthropic account ban. Cost attribution must stay clean and separable.
- **Cost-conscious by default.** The user deliberately stays on Anthropic-direct rather than routing through OpenRouter to avoid proliferating charges, and uses Gemini's free embedding tier. Spend where it makes sense, but no accidental fan-out of paid calls.
- **Clean state, no cruft.** Strong preference for direct action over hedging and for leaving nothing stale behind ("delete the key from `.env`, don't comment it out"). The system should prune, not accumulate dead files, commented config, or orphaned cache.
- **High autonomy.** Because the VPS is isolated and single-user, the agent runs with `approvals.mode=auto` and acts without round-trips for routine ingest/retrieve/summarise work.
- **Permanent, git-tracked, cloneable, browsable + searchable.** A standing frustration with the proto-vault was uploads landing in *temporary caches*. The replacement must be a **cloneable git repo of real files** that is browsable by hand **and** semantically searchable — both, not either.
- **Model routing preference.** Sonnet for heavy multi-step infrastructure work; lighter models acceptable for trivial answers, delegating hard technical work to Sonnet subagents (see §4 and §5 for the reconciled single target default).

---

## 3. System Architecture

One agent, one canonical vault, one dedicated git repo, two-way synced to the workstation, with Hindsight indexing the vault and cron driving summaries. The diagram below is the **single consolidated architecture view** for this document; later sections reference it rather than redraw it.

```
                          ISOLATED VPS
  ┌──────────────────────────────────────────────────────────────┐
  │                                                                │
  │   INTERFACES              HERMES AGENT            INDEX        │
  │  ┌───────────┐         ┌─────────────────┐                    │
  │  │ Slack DM  │────────▶│  capture /      │                    │
  │  │ (primary, │◀────────│  retrieve /     │                    │
  │  │  U0B5ZP…) │ answers+ │  summarise loop │   reindex /        │
  │  └───────────┘ obs://   │                 │───recall──┐        │
  │  ┌───────────┐  links   │  Anthropic API  │           ▼        │
  │  │ MCP        │────────▶│  (pay-as-you-go,│   ┌───────────────┐│
  │  │ (Claude    │◀────────│   NOT Claude Max)│  │   HINDSIGHT   ││
  │  │  Code/.ai) │  paths+  │                 │  │ local_embedded ││
  │  └───────────┘ wikilinks │  Gemini embeds  │  │ Gemini extract ││
  │                          └───────┬─────────┘  │ bank_id=hermes ││
  │   SCHEDULER                      │            │ local Postgres ││
  │  ┌───────────┐  fire prompt      │ pull-before-write          ││
  │  │ Hermes    │──────────────────▶│ commit+push-after-ingest   ││
  │  │ cron      │  daily 07:00      │            └───────▲───────┘│
  │  │ (Europe/  │  weekly Mon 07:00 │                    │        │
  │  │  London)  │                   ▼                rebuilds     │
  │  └───────────┘        ┌──────────────────────┐      from      │
  │                       │   CANONICAL VAULT     │──────vault─────┘│
  │                       │  raw/ entities/       │                 │
  │                       │  concepts/ comparisons/│                │
  │                       │  queries/ + Home/index │                │
  │                       │  SCHEMA.md log.md      │                │
  │                       │  (markdown + assets)   │                │
  │                       └───────────┬───────────┘                 │
  └───────────────────────────────────┼─────────────────────────────┘
                                       │ git pull --rebase / push
                                       ▼
                        ┌──────────────────────────┐
                        │  DEDICATED PRIVATE        │
                        │  GitHub repo (pkm-vault,  │
                        │  vault only, separate from│
                        │  the hermes-agent config  │
                        │  repo)                    │
                        └───────────┬──────────────┘
                                    │ Obsidian Git plugin
                                    │ (scheduled auto pull/commit/push)
                                    ▼
                        ┌──────────────────────────┐
                        │  WORKSTATION — Obsidian   │
                        │  human browses, edits,    │
                        │  clicks obsidian:// links  │
                        └──────────────────────────┘
```

**Reading the diagram:**

- **Two writers, one store.** The VPS agent and the workstation Obsidian both edit the *same* vault, reconciled through the **dedicated private GitHub repo** (`pkm-vault`). The agent `pull --rebase`s before writing and `commit`+`push`es after each ingest; Obsidian Git does scheduled auto pull/commit/push the other way. Collisions are rare because `raw/` is immutable and the layout is one-file-per-topic (see §12 for the detailed protocol).
- **Index hangs off the store.** Hindsight indexes curated vault pages (not chatter) and is **rebuildable from the vault** via a reindex job — the dashed "rebuilds from vault" arrow. If the index is lost or drifts, it is regenerated; nothing canonical is at risk.
- **`state.db` is deliberately absent from the canonical path.** Hermes' SQLite session DB is transient working memory only and is **not** drawn as a knowledge store. This is the architectural retirement of the old "the DB is the truth" model.
- **Cron is a driver, not a store.** The Hermes cron scheduler fires summary prompts on Europe/London time; the agent reads the *vault* to compose them and delivers to Slack.

### Interfaces

#### Slack DM — primary

Conversational capture, retrieval, and summary delivery all happen in the user's Slack DM (`D0B61LKA3NV`). This is the everyday surface: paste a URL, drop a PDF, snap a photo, type a thought or a TODO, or ask a question.

Access posture (from `config.yaml`):

```yaml
slack:
  require_mention: true        # bot only acts when addressed
approvals:
  mode: auto                   # no approval round-trips (isolated, single-user)
timezone: Europe/London
```

- `require_mention: true` keeps the bot from reacting to unrelated traffic. **In a 1:1 DM every message is implicitly directed at the bot**, so capture and retrieval feel natural there; in any shared channel the user must @-mention. The DM is the recommended primary surface precisely because it sidesteps mention friction while staying scoped to one trusted user.
- **Allowed-user posture:** the workspace is private to the single user (`SLACK_ALLOWED_USERS=U0B5ZP301H8`). `allowed_channels` is left unset (DM-scoped usage); because the box is isolated with `approvals.mode=auto`, the trust boundary is "this Slack workspace = this one user." Any future shared channel should be added to `allowed_channels` explicitly rather than opening the bot up broadly.
- **Slack-native strengths used here:** inline file upload (photos, PDFs) for capture; clickable links and `mrkdwn` formatting for retrieval; scheduled message delivery for summaries.

#### MCP — secondary (Claude Code / claude.ai)

The same capture and retrieval capabilities are exposed over MCP for use *inside* a Claude Code session or claude.ai desktop — ideal for filing a technical note or searching the vault mid-task without leaving the editor.

- **Best at:** text in, text out — search results, page contents, metadata, and **vault-relative paths + wikilinks** the calling Claude can render or follow.
- **Weaker at:** pushing binary files back to the user (no Slack-style upload). MCP retrieval therefore leans on `obsidian://` links and paths (§8) rather than attachments.
- **Deployment:** local stdio MCP for Claude Code (no network surface). If exposed remotely for claude.ai it must sit behind authenticated transport (Cloudflare Tunnel with auth), never bare HTTP (§14). Note: an `obsidian://` link only opens if the user's Obsidian is installed on the same device as the MCP host — from claude.ai web on a foreign machine the vault-relative path + wikilink are the usable handles.

| Capability | Slack DM (primary) | MCP (secondary) |
|------------|--------------------|-----------------|
| Conversational capture | ✅ native, incl. file/photo upload | ✅ text + path-referenced files |
| Natural-language retrieval | ✅ | ✅ |
| Return original document | ✅ `obsidian://` link **+** Slack upload fallback | ✅ `obsidian://` link + path/wikilink (no upload) |
| Proactive summaries | ✅ delivered here | ➖ on-demand only |
| Link rendering | `mrkdwn` clickable links | Markdown links / raw paths the host renders |

---

## 4. The Hermes Platform

A self-contained primer on the platform Hermes provides, condensed from the as-built `hermes-agent` repo and the prior planning notes. For this PKM system Hermes is the **execution substrate** — the conversation loop, the gateways, the scheduler, and the tool runtime. It is **not** the knowledge store; that role belongs to the Obsidian vault.

**What it is.** Hermes Agent is an open-source, self-improving AI agent framework from Nous Research (MIT license, Python 3.11+). It is a long-running, multi-channel agent runtime: a conversation loop that drives an LLM through tool calls, persists every turn to a local SQLite session DB, exposes itself over chat gateways (Slack, Discord, etc.) and an MCP/API server, and schedules its own work via a built-in cron scheduler. It ships 70+ self-registering tools, multi-agent delegation, a pluggable memory subsystem, and a community skill hub.

| Property | Value |
| --- | --- |
| Vendor / license | Nous Research / MIT |
| Language | Python 3.11+ (Node 20+ optional: TUI, browser tools, WhatsApp) |
| Repo | `github.com/NousResearch/hermes-agent` |
| Package manager | `uv` |
| Framework default model | `anthropic/claude-opus-4.6` (this deployment overrides it — see §5) |
| Agent home | `~/.hermes/` |

**Install options.**

```bash
# One-liner (production)
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# Dev install (editable, all extras)
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent && uv venv venv --python 3.11
export VIRTUAL_ENV="$(pwd)/venv"
uv pip install -e ".[all,dev]"

# Docker: docker-compose provides gateway (agent) + dashboard (web UI),
# volume ~/.hermes:/opt/data, network mode host
docker compose up -d
```

Post-install: `hermes setup` (interactive wizard), `hermes doctor` (health check), `hermes model` (runtime model switch), `hermes --tui` (Ink TUI), `hermes skills install` (community skills).

**Config model — two files under the agent home.** Hermes separates declarative config from secrets:

- `~/.hermes/config.yaml` — the full agent configuration (model defaults, toolsets, memory provider, gateway behaviour, approvals, cron, delegation, timezone, …). Seeded from `cli-config.yaml.example`. The schema is large (`_config_version: 23` in this deployment).
- `~/.hermes/.env` — all secrets and many feature flags (~350+ documented vars). Seeded from `.env.example`. **Never committed** (gitignored).

The agent home also contains runtime/state subtrees: `cron/`, `sessions/`, `logs/`, `memories/`, `skills/`, plus provider-specific dirs such as `hindsight/`.

**LLM providers.** ~29 first-class providers (Anthropic, OpenAI, OpenRouter, Google Gemini, DeepSeek, xAI, Nous, Bedrock, Ollama, Azure Foundry, …) plus any OpenAI-compatible endpoint via the "Custom" provider. Selection is per-context: a primary `model`/`provider`, an optional `fallback_model` for failover on 429/529/503, plus independent overrides for `delegation`, each `auxiliary.*` helper task (vision, web_extract, compression, curator, title_generation, …), and per-cron-job. Keys live in `.env` (`ANTHROPIC_API_KEY`, `GOOGLE_GEMINI_API_KEY`, `OPENROUTER_API_KEY`, …).

**Gateway platforms.** 15+ messaging adapters: Telegram, Discord, **Slack**, WhatsApp, Signal, Email, SMS, Matrix, Teams, DingTalk, Feishu/Lark, WeCom, Home Assistant, QQ Bot, BlueBubbles (iMessage), plus a generic webhook and an API/MCP server. Each platform binds a `platform_toolsets` group (e.g. Slack → `hermes-slack`) and supports `require_mention`, allow-lists, and per-channel prompts.

**Tool categories.** Terminal execution (7 backends: local/Docker/SSH/Singularity/Modal/Daytona/PTY), file ops, browser automation (Playwright/Camofox/CDP), web search/extract, sandboxed code execution, vision, image/video/TTS generation, memory, skills management, cron, delegation, kanban, MCP integration, computer-use, and platform-native tools. Tools self-register into ~30 named toolset groups; groups can be enabled/disabled per platform and per delegate.

**Cron.** A built-in scheduler (`cron/jobs.json`) runs prompts on cron expressions in the configured `timezone`. Each job can pin its own `model`/`provider`, target toolsets, and a `deliver` sink (e.g. `slack:<channel-id>` or `origin`). Used here for daily/weekly summaries and the nightly reindex.

**Delegation.** The agent can spawn subagents (single or parallel batch) under an orchestrator, with its own `delegation.model`, concurrency caps (`max_concurrent_children`), and spawn-depth limits — used to push heavy multi-step work onto a stronger model than a cheaper default.

**Memory providers.** Memory is pluggable (`memory.provider`). Options include the bundled summary memory and external plugins (honcho, mem0, **hindsight**). Hindsight adds semantic recall over an embedded vector index. Orthogonally, Hermes maintains compressed markdown summaries (`memories/MEMORY.md`, `memories/USER.md`) that are injected into the system prompt each turn — these are *prompt-context snapshots*, not the store.

**Security features.** Dangerous-command detection with an approval flow (`approvals.mode`), a cron prompt-injection scanner, a write deny-list with symlink-bypass prevention, a skills guard plus the bundled **Tirith** security scanner (`security.tirith_enabled`, binary at `bin/tirith`), tool-loop guardrails (soft warnings → hard stops), container hardening, and API-key stripping from child environments.

---

## 5. As-Built Today: Inventory and What Must Change

This is the concrete "starting state" the migration plan (§16) transforms. Paths below are relative to the agent home (the repo root on the VPS, mirrored here at `/workspaces/hermes-agent/`).

### Inventory of this deployment

**Identity, interfaces, autonomy.**

- Single user: Giles Knap (`gilesknap@gmail.com`; Slack `U0B5ZP301H8`, DM channel `D0B61LKA3NV`). Slack DM via App Home is the primary interface; `slack.require_mention: true`; `SLACK_ALLOWED_USERS=U0B5ZP301H8`.
- Gateway runs as systemd `hermes-gateway.service`, enabled at boot — generated by `hermes gateway install` and controlled via `hermes gateway start|stop|restart`. **The target keeps this tool-generated name** (`hermes-gateway.service`) rather than introducing a hand-rolled `hermes-pkm.service`, so there is exactly one gateway unit and no fabricated `ExecStart` (§15 Phase F).
- `approvals.mode: auto` and all approval gates disabled — high autonomy on an isolated VPS.
- `timezone: Europe/London`.
- MCP server is referenced as the secondary interface (Claude Code / claude.ai) but `processes.json` is empty (`[]`) — no MCP process is currently registered/running.

**Models and providers (`config.yaml`).**

| Setting | As-built value | Note |
| --- | --- | --- |
| `model.default` / `provider` | `claude-sonnet-4-20250514` / `anthropic` | Direct Anthropic, pay-as-you-go |
| `delegation.model` | `claude-sonnet-4-20250514` / `anthropic` | `max_concurrent_children: 2`, `max_spawn_depth: 1` |
| `auxiliary.compression.model` | `claude-sonnet-4-20250514` | |
| `auxiliary.curator.model` | (LLM helper task, timeout 600s) | distinct from the top-level `curator` block |
| `curator` (top-level) | `interval_hours: 168`, stale/archive policy | the **skills** curator, not a vault job |
| `memory.provider` | `hindsight` | `memory_enabled: true`, char limits 2200 / 1375 |
| `web.search_backend` / `extract_backend` | `exa` / `firecrawl` | both keys configured and tested |
| `security` | `tirith_enabled: true`, `redact_secrets: true`, `tirith_fail_open: true`, `allow_private_urls: false` | |

The user deliberately keeps two separate Anthropic billing streams: the Claude Max (claude.ai) subscription for interactive use, and a pay-as-you-go API key for Hermes. The agent must use the API key only.

**Hindsight memory (`hindsight/config.json`).**

```json
{ "mode": "local_embedded", "bank_id": "hermes",
  "llm_provider": "gemini", "llm_model": "gemini-2.0-flash-exp",
  "auto_recall": true, "auto_retain": true, "memory_mode": "hybrid",
  "recall_budget": "mid", "recall_max_tokens": 4096,
  "retain_async": true, "retain_every_n_turns": 1,
  "retain_context": "conversation between Hermes Agent and the User" }
```

Critically, `auto_retain: true` + `retain_every_n_turns: 1` + the conversational `retain_context` mean Hindsight is currently indexing **raw conversation turns**, so recall returns chatter rather than curated knowledge.

**Session store — `state.db` (the central as-built fact).** The live `state.db` is gitignored at the home root; a committed snapshot lives at `db-snapshots/state.db` (≈14.5 MB). It is a **conversation log, not a structured document store.** Verified contents:

- `messages`: **1,519 rows** — 737 `assistant`, 590 `tool`, 186 `user`, 6 `session_meta`.
- `sessions`: **41 rows** (39 distinct session IDs appear in `messages`).
- Full-text search is FTS5 over a single `content` column, via `messages_fts` (standard tokenizer) and `messages_fts_trigram` (trigram), kept in sync by insert/update/delete triggers.
- `messages` columns: `id, session_id, role, content, tool_call_id, tool_calls, tool_name, timestamp, token_count, finish_reason, reasoning*, platform_message_id, observed`.

There is no documents table, no per-fact rows, no image table — "knowledge" is reconstructed by keyword-searching the chat transcript. `kanban.db` is also snapshotted (≈104 KB).

**Documents proto-vault (`documents/`) — git-tracked.** `.gitignore` explicitly *un-ignores* `documents/**` and `db-snapshots/*.db` while ignoring secrets, caches (`image_cache/`, `audio_cache/`), `sessions/`, `logs/`, and live `*.db`. The proto-vault is shallow and image-only:

```
documents/
├── README.md
└── images/
    ├── img_e4a408e064c4.png   (1.8 MB)  + img_e4a408e064c4.md   (sidecar)
    ├── img_ed814b035583.png   (1.7 MB)  + img_ed814b035583.md   (sidecar)
    └── img_85e18fa08300.jpeg  (269 KB)  ← NO sidecar (orphan)
```

The README advertises `pdfs/ notes/ references/` subdirs, but only `images/` exists. Each image follows the **binary + sidecar `.md`** pattern: a descriptive markdown file (`# Title`, `**File:**`, `**Tags:**`, Description, Context) that *names* the image but does not embed it. The same metadata is duplicated in **three** places: the sidecar `.md`, the Slack chat messages in `state.db`, and Hindsight facts (and partly again in `memories/MEMORY.md`, which contains the "Technical Diagrams" and "Soldering Zoo Kit" entries). The runtime copy lives separately at `~/.hermes/image_cache/img_<uuid>.<ext>` (gitignored); `documents/images/` is the committed copy.

**Persona — `SOUL.md` (carries the claim to retire).** The current persona is a PKM "second brain" with explicit ingest/retrieve/TODO procedures, but it hard-codes the wrong source-of-truth model:

> "**Hindsight DB** (the real backend) … Stores EVERYTHING you ingest … the markdown files are just convenient snapshots — **the DB is the truth**." (the *How Memory Works* section near the top of `SOUL.md`; locate by content with `grep -n "DB is the truth"` rather than a line number)

It instructs the agent to persist via `memory(action='add')` and retrieve via `session_search()` — i.e. it treats the transient session DB as the knowledge base. This is the exact claim the new architecture retires.

**Cron jobs (`cron/jobs.json`).** Three jobs, all delivering to Slack DM `D0B61LKA3NV`:

| Job | Schedule (Europe/London) | Model/provider | Status |
| --- | --- | --- | --- |
| `daily-summary` | `0 7 * * *` | `claude-sonnet-4` / `anthropic` | `last_status: error` — `HTTP 404: model: claude-sonnet-4` |
| `weekly-summary` | `0 7 * * 1` | `gemini-pro` / `gemini` | never run |
| `Hermes repo review reminder` | `0 9 * * 6` (one-shot) | default | scheduled |

The summary jobs reference model IDs (`claude-sonnet-4`, `gemini-pro`) that do not resolve, so the daily summary 404s and the weekly has never fired.

**Skills inventory (`skills/`).** **24 skill groups, 96 `SKILL.md` files** installed. Most relevant to this spec:

- `research/llm-wiki` (v2.1.0, MIT) — implements Karpathy's three-layer LLM Wiki: `SCHEMA.md` + `index.md` + `log.md`; Layer-1 immutable `raw/{articles,papers,transcripts,assets}/`; Layer-2 curated `entities/ concepts/ comparisons/ queries/`. Defines the YAML frontmatter (`title, created, updated, type, tags, sources`), the `[[wikilinks]]` convention (≥2 outbound per page), provenance markers, and a mandatory session-start orientation (read SCHEMA → index → recent log). **This skill is the blueprint for the canonical vault.**
- `productivity/personal-knowledge-management` (v1.0.0) — current PKM operating doc; still describes the store as "SQLite + FTS5 + Hindsight," i.e. the to-be-retired model.
- `note-taking/obsidian` — filesystem-first vault ops (read/list/search/create/append/wikilink) via file tools; vault path from `OBSIDIAN_VAULT_PATH` (fallback `~/Documents/Obsidian Vault`); warns that file tools do not expand shell variables, so the path must be resolved to a concrete absolute path first.
- Bespoke `devops/` skills authored for this deployment: `hermes-pkm-ops` (persona, cron, MCP, git-backed backup, plus references including `pkm-ingestion-pitfall.md`, `memory-architecture.md`, `image-filesystem-layout.md`, `document-storage-architecture.md`), `hermes-pkm-embeddings` (FTS5-vs-embeddings tradeoffs), `hermes-messaging-gateway`, `hermes-vps-setup`. These encode the *old* "Hindsight DB = knowledge base" mental model and must be re-aligned to "vault = source of truth, Hindsight = rebuildable index."

A documented operational pitfall is worth carrying forward: the **"acknowledge-without-persist"** bug — models conflate *summarise* with *store* and confirm an ingest without ever writing it. The fix (validated in `pkm-ingestion-pitfall.md`) is to make the persist step an explicit numbered action, not a parenthetical. Under the new architecture this same discipline applies to the *vault write* (pull → write file → commit/push), and the "did it actually land?" validation test is retained.

**Repository.** The agent home is a git repo whose `origin` is `github.com/gilesknap/hermes-agent` — i.e. the config-backup repo, into which `documents/` and `db-snapshots/` are currently committed alongside config, skills, and memories. Backups are taken by `backup.sh` (under the pkm-ops skill) as scheduled "backup `<timestamp>`" commits. In the target runbook this as-built `backup.sh` is promoted to a delivered artefact and renamed **`config-backup.sh`** (§12, §15 Phase G — same role, full body provided there); the two names refer to the same config-backup mechanism.

### What must change (delta to the target architecture)

The table maps each as-built reality to the required change; §16 performs the step-by-step edits.

| # | As-built state | Required change | Rationale |
| --- | --- | --- | --- |
| 1 | `SOUL.md` asserts "**the DB is the truth**"; ingest = `memory(action='add')`, retrieve = `session_search()`. | **Retire DB-as-truth.** Rewrite the persona so the **Obsidian vault is canonical**; `state.db` is transient working memory only; ingest = *write a curated vault page* (pull → write → commit/push); retrieve = *return vault pages + `obsidian://` deep links* (persona in §7). | The session DB is a 1,519-message transcript with FTS5 over one `content` column — provably not a document store. Treating it as the KB causes loss/confabulation. |
| 2 | `documents/` is a flat, image-only proto-vault (3 images, 2 sidecars, 1 orphan) committed inside the config repo. | **Migrate `documents/` into the new canonical vault** under the llm-wiki layout: image binaries → `raw/assets/`; their knowledge content → curated pages in `concepts/`/`entities/` with proper frontmatter; create `SCHEMA.md`, `index.md`, `log.md`; back-fill the orphan `img_85e18fa08300.jpeg`. | One-file-per-topic, cross-linked pages — not a loose image dump duplicated across DB + sidecar + Hindsight. |
| 3 | Vault content lives in `gilesknap/hermes-agent` (the config-backup repo). | **Split the vault into its own dedicated private GitHub repo** (`pkm-vault`), separate from `hermes-agent`. Stop tracking `documents/`/vault content in the config repo. | The vault needs independent two-way git sync (Obsidian Git plugin on the workstation; agent pull-before-write / commit-after-ingest) without entangling agent config and DB snapshots. |
| 4 | Hindsight `auto_retain: true`, `retain_every_n_turns: 1`, conversational `retain_context` — indexes raw chat. | **Re-point Hindsight to index curated vault content**, not transcript: set `auto_retain: false`, point `retain_context` at filed pages, and add a **reindex job that rebuilds Hindsight from the vault**. Keep `provider: hindsight`, `local_embedded`, Gemini, `bank_id: hermes`. | Recall must surface filed pages/facts, not chatter. The index becomes a rebuildable projection of the vault (§9). |
| 5 | Image = binary + separate descriptive sidecar `.md`; metadata duplicated across sidecar, chat, and Hindsight. | **Merge the sidecar pattern into curated pages that embed their images** via Obsidian wiki-embeds (`![[asset.ext]]`); binaries in `raw/assets/`; no base64. Eliminate the duplicate sidecar + chat + Hindsight copies in favour of one curated page. | Single source per topic; the page both describes *and* shows the image; removes triplicated metadata. |
| 6 | Cron `daily-summary`/`weekly-summary` reference non-resolving models (`claude-sonnet-4` 404; `gemini-pro` never ran). | **Repoint cron jobs to a resolvable model id** — `claude-sonnet-4-6` (verify with `hermes model`; dated `claude-sonnet-4-20250514` is the proven fallback) — and to the vault-backed summary flow (read vault dashboards/Bases, not the DB). Keep daily 07:00 + weekly Mon 07:00 Europe/London → Slack. | The headline proactive feature is currently broken; it must read from the canonical vault (§11). |
| 7 | `config.yaml` default is `claude-sonnet-4-20250514`; documented policy is "Haiku-simple / Sonnet-delegate"; `processes.json` empty (no MCP). | **Declare one target default** — `anthropic/claude-sonnet-4-6` — and align delegation + cron to it; route trivial answers to a lighter model only via explicit per-call/delegate overrides. **Stand up the MCP server** as the documented secondary interface — the native `hermes mcp serve` stdio server registered in `~/.claude/settings.json`, exposing `pkm_ingest`/`pkm_search`/`pkm_todos`/`pkm_recent` (§15 Phase F.1); it runs alongside the always-on Slack gateway and is launched per-session by the MCP client (no second daemon). Ensure the agent uses only the pay-as-you-go API key, never the Claude Max OAuth token. | Running config must match the spec's interface/model decisions; billing streams stay separated to avoid the OAuth-token ban risk. |
| 8 | Bespoke `devops/hermes-pkm-*` skills + `productivity/personal-knowledge-management` still teach "Hindsight DB = knowledge base." | **Re-align the operating skills** to "vault = source of truth, Hindsight = rebuildable index," promoting `research/llm-wiki` + `note-taking/obsidian` as the primary operating skills; carry forward the acknowledge-without-persist discipline applied to vault writes. | The agent's own instructions must stop pointing at the retired model. |

#### Reconciled model strategy (resolves the model-default drift)

There were three conflicting model claims (the framework default `claude-opus-4.6`, the live config `claude-sonnet-4-20250514`, and a documented "Haiku-default" policy in `USER.md`). This spec declares **one** target:

- **`model.default = anthropic/claude-sonnet-4-6`** — the single default for the gateway, delegation, and cron, **subject to verification** (see below). It is attested running in this deployment (it appears as `model=claude-sonnet-4-6` in the as-built gateway logs), and standardising on one id keeps cron and config identical. **The proven-good fallback is the dated pin `claude-sonnet-4-20250514`** (the current as-built working value). Note that the only failure actually observed is an *unresolved alias* (`claude-sonnet-4` → HTTP 404), so confirm `-4-6` resolves with `hermes model` before relying on it; if it does not, use the dated id everywhere. Do **not** assume the daily-summary 404 is "resolved" until `-4-6` is verified — the 404 fix is simply moving off the bare `claude-sonnet-4`/`gemini-pro` ids to a resolvable one.
- **Lighter-model routing is opt-in, not the default**: trivial answers may use a cheaper model only via an explicit per-call or per-delegate override; there is no global Haiku default. This reconciles the `USER.md` "Haiku-simple / Sonnet-delegate" intent with a config that actually resolves, and keeps heavy multi-step work on Sonnet.
- The agent authenticates with the pay-as-you-go `ANTHROPIC_API_KEY` only — never the Claude Max OAuth token (§14).

#### Disposition of as-built artefacts (resolves several open issues)

- **`db-snapshots/{state.db,kanban.db}`** remain in the **config-backup repo** (`hermes-agent`), not the vault repo. They are reclassified as backups of *transient working memory* (in-flight sessions + Kanban), not knowledge. No knowledge needs salvaging from the 1,519-message transcript before it is treated as disposable, because the migration (§16) lifts the only durable content — the three images and their descriptions — into the vault first. If the repo later strains under the ~14.5 MB snapshot, graduate it to Git LFS; do not move it into `pkm-vault`.
- **Carry-over memories.** `memories/MEMORY.md` / `USER.md` entries that duplicate *ingested content* (Technical Diagrams, Soldering Zoo Kit) are migrated into the vault as curated pages and then trimmed from the prompt-context summaries, so the same facts are not triplicated. Operational facts (model policy, schedules, preferences) stay as Hermes prompt-context summaries.
- **Curator naming.** Neither `auxiliary.curator` (an LLM helper task) nor the top-level `curator` block (the **skills** curator: `interval_hours: 168`, stale/archive policy) is repurposed for vault work. The vault reindex (§9) and lint (§13) are their **own** dedicated cron jobs to avoid confusion.

---

## 6. The PKM Vault (Canonical Store)

This section specifies the canonical knowledge store: a single Obsidian vault of markdown files, organised as a hybrid of Karpathy's three-layer LLM Wiki (raw sources → curated pages → schema) and PKM life-admin domains. The vault is the source of truth. Hindsight is a rebuildable index over it; `state.db` is transient working memory. Nothing here treats the SQLite session DB or chat logs as knowledge.

> **Retire the legacy claim.** `SOUL.md` (the sentence currently reading "The markdown files are just convenient snapshots — the DB is the truth", in the *How Memory Works* section near the top of the file) must be inverted: **the vault is the truth; the DB and the Hindsight index are disposable projections of it.** Locate it by content (`grep -n "DB is the truth" ~/.hermes/SOUL.md`) rather than by line number, since the file is rewritten in full by §7. Any prose elsewhere that says otherwise is superseded by this section.

### Vault identity and on-disk location

| Property | Value |
|---|---|
| Vault name (for `obsidian://` links and `OBSIDIAN_VAULT_NAME`) | `pkm-vault` |
| VPS path | `/opt/pkm-vault`. Set `PKM_VAULT=/opt/pkm-vault` **and, because the skills do NOT auto-alias it,** set `WIKI_PATH=/opt/pkm-vault` (read by the llm-wiki skill, default `~/wiki`) and `OBSIDIAN_VAULT_PATH=/opt/pkm-vault` (read by the obsidian skill, default `~/Documents/Obsidian Vault`) **each explicitly** in `~/.hermes/.env` and the gateway/MCP env — otherwise those skills fall back to their defaults and write to the wrong place (§15 Phase F). |
| Git remote | dedicated **private** repo `github.com/<owner>/pkm-vault` — separate from the `hermes-agent` config-backup repo |
| Workstation sync | Obsidian + **Obsidian Git** community plugin (scheduled pull/commit/push) |
| Attachment folder (Obsidian setting) | `raw/assets` |
| New/default file location (Obsidian setting) | `raw/articles` (raw capture lands here first; the agent files curated pages from it) |

The vault root **is** the git root and **is** the Obsidian vault root — no nesting mismatch. The vault **name** (`pkm-vault`) must match the Obsidian vault registration on every device exactly, or `obsidian://` deep links will not resolve on that device; the agent stores this one string as `OBSIDIAN_VAULT_NAME` and uses it verbatim in every reply. The file tools do not expand shell variables, so the agent resolves `PKM_VAULT` to the concrete absolute path `/opt/pkm-vault` before any file operation.

> **Resolution of the vault-name collision:** earlier drafts used both `hermes-vault` and `pkm-vault`. This spec standardises on **`pkm-vault`** everywhere — it matches the git repo name, the Obsidian vault folder name, and the worked deep-link examples — so the repo, the folder, and the `vault=` link parameter are the same string on all devices.

### Hybrid folder tree

The knowledge core uses the llm-wiki layers verbatim. Life-admin is **not** a rival folder tree — Actions/Media/Memories are wiki pages distinguished by a frontmatter `type` field and surfaced as Obsidian **Bases** dashboards (dynamic views). The only additional top-level life-admin folders are `inbox/` (the ambiguous-capture fallback) and `people/` (a first-class entity sub-domain that life-admin pages link into).

```
pkm-vault/
├── index.md              # HOME landing page: unifies knowledge + life-admin, embeds Bases
├── SCHEMA.md             # Conventions, frontmatter contract, tag taxonomy, thresholds
├── log.md                # Append-only action log (rotated yearly + at 500 entries)
│
├── raw/                  # LAYER 1 — immutable sources (agent reads, never edits)
│   ├── articles/         #   web clippings (Firecrawl/Exa extract -> markdown)
│   ├── papers/           #   papers/arxiv: extracted <slug>.md + the source <slug>.pdf alongside it
│   ├── transcripts/      #   meeting notes, voice memos, interview transcripts
│   └── assets/           #   binary images/diagrams embedded by curated pages (NOT paper PDFs)
│
├── entities/             # LAYER 2 — people, orgs, products, models, beamlines, devices
├── concepts/             # LAYER 2 — topics, techniques, how-tos, reference explainers
├── comparisons/          # LAYER 2 — side-by-side analyses (table-first)
├── queries/              # LAYER 2 — filed answers worth keeping
│
├── actions/              # life-admin: TODOs (type: action) — one file per task
├── media/                # life-admin: to-consume backlog (type: media)
├── memories/             # life-admin: personal memories/milestones (type: memory)
├── people/               # entity sub-domain that memories/actions link into
│
├── _bases/               # Obsidian Bases definition files (.base) — dashboards
│   ├── home.base
│   ├── actions.base
│   ├── media.base
│   ├── memories.base
│   └── inbox.base
│
├── _meta/                # navigation aids for large vaults (topic-map.md when index > 200)
├── _archive/             # superseded pages (mirrors original path; removed from index)
├── inbox/                # ambiguous captures awaiting classification (type: inbox)
└── .obsidian/            # plugin config: metadata-menu presets, obsidian-git, Bases
```

Rationale for keeping `actions/`/`media/`/`memories/`/`people/` as real folders rather than collapsing everything by `type` alone: they have a different lifecycle from knowledge pages (status churn, due dates, completion) and one-folder-per-domain keeps the Obsidian "new note here" workflow and the agent's pull-before-write collision surface predictable. They are still **pages with frontmatter `type`**, so Bases can union them with knowledge pages on the Home page — the folder is an implementation detail, the `type` field is the contract. Because they are real folders, life-admin pages have real vault-relative paths (`actions/…`, `media/…`, `memories/…`) and therefore real `obsidian://` deep links.

The underscore-prefixed dirs (`_bases/`, `_meta/`, `_archive/`) are structural, not knowledge: they are **excluded from the Hindsight reindex** (§9) and from global Bases `file.ext == "md"` knowledge views; `_archive/` is dropped from `index.md`.

### Frontmatter schema (knowledge + life-admin in one contract)

Every `.md` page begins with a YAML block. Fields split into **common** (all pages), **knowledge** (`type: entity|concept|comparison|query|summary`), and **life-admin** (`type: action|media|memory|inbox`). `raw/` files carry their own minimal block (see §7).

Common (required on every page):

```yaml
---
title: Human Readable Title
type: entity            # entity|concept|comparison|query|summary|action|media|memory|inbox
created: 2026-05-30
updated: 2026-05-30
source: slack           # slack | mcp | web | manual | cron
tags: [kebab-case, from-taxonomy]
---
```

Knowledge pages add (all optional except where noted):

| Field | Type | Meaning |
|---|---|---|
| `sources` | list | `[raw/articles/foo.md]` — raw files this page synthesises (required if any raw exists) |
| `confidence` | `high\|medium\|low` | how well-supported; default unset = treat as medium. Lint flags `low` and single-source-without-confidence |
| `contested` | `true` | page has unresolved contradictions; surfaced by lint |
| `contradictions` | list | page slugs this one conflicts with |
| `aliases` | list | alternate names (Obsidian alias resolution for wikilinks) |

Life-admin pages add:

| Field | Applies to | Type / values | Meaning |
|---|---|---|---|
| `status` | action, media | `todo\|in_progress\|done\|completed\|cancelled` (action); `to_consume\|consuming\|consumed` (media) | Metadata Menu preset-backed |
| `priority` | action, media | `1 - Urgent\|2 - High\|3 - Medium\|4 - Low` | Metadata Menu preset-backed |
| `due_date` | action | `YYYY-MM-DD` or `YYYY-MM-DD HH:MM` or empty | for sort/filter and daily briefing |
| `recurrence` | action | `none\|daily\|weekly\|monthly\|yearly` (or RRULE-lite string) | repeating tasks; agent re-opens on completion |
| `project` | action | wikilink `"[[project-slug]]"` or empty | links task to its concept/entity page |
| `media_type` | media | `book\|film\|tv\|podcast\|article\|video\|music` | Metadata Menu preset-backed |
| `creator` | media | string | author/director/artist |
| `url` | media | URL or empty | link to the item |
| `people` | memory, action | list of names (each a `[[people/...]]` link where known) | who was involved |
| `location` | memory | string | where it happened |
| `memory_date` | memory | `YYYY-MM-DD` or empty | when it happened, if different from `created` |

Notes:
- `category` from the old 2ndBrain schema is **dropped** — `type` plus folder replace it. Mapping the two legacy 2ndBrain categories: **`Reference` → `concepts/`** (reference explainers are concept pages), and **`Projects` → project *pages* live in `entities/`** (a project is a named thing; use `concepts/` only if it is better modelled as a body of work than a named entity). An action's `project:` is a wikilink to that `entities/` (or `concepts/`) page — e.g. `project: "[[home-maintenance]]"` points at `entities/home-maintenance.md`. There is no separate `projects/` folder.
- `tokens_used` (old Gemini accounting field) is **dropped** from the page contract; cost telemetry lives in `log.md`/`state.db`, not in knowledge.
- The Metadata Menu preset config (`status`, `priority`, `media_type` dropdowns) is preserved verbatim so in-Obsidian editing offers the same controlled vocabularies. Ship it at `.obsidian/plugins/metadata-menu/data.json`.

### File-naming conventions

| Layer | Convention | Example |
|---|---|---|
| Curated pages (entities/concepts/comparisons/queries) | lowercase, hyphenated, one-topic-per-file, no dates in name | `entities/program-motion-controller.md` |
| Life-admin pages | same lowercase-hyphen rule; dates live in frontmatter only | `actions/fix-garden-fence.md` |
| `people/` | lowercase hyphen of the person's name | `people/jane-doe.md` |
| Raw sources | descriptive, hyphenated, source-typed; keep a date suffix only when it disambiguates | `raw/articles/karpathy-llm-wiki-2026.md`, `raw/papers/attention-is-all-you-need.md` |
| Binary assets | `raw/assets/<slug>-<shorthash>.<ext>` — stable, collision-proof, descriptive | `raw/assets/motor-control-diagram-e4a408.png` |

This deliberately abandons the old 2ndBrain `Attachments/20260207_113000_photo.png` timestamp-prefix scheme: timestamps in filenames are noise once frontmatter carries dates, and a descriptive slug makes wiki-embeds self-documenting and survivable across re-ingests.

### Images: embed-and-describe, not sidecar

Today's proto-vault pattern (`documents/images/img_e4a408e064c4.png` + a separate `img_e4a408e064c4.md` describing it) is **replaced**. New rule:

1. Binary lands in `raw/assets/` with a descriptive slug.
2. The **curated page** (an entity/concept/memory page) **both embeds and describes** the image inline using an Obsidian wiki-embed: `![[motor-control-diagram-e4a408.png]]`.
3. No separate descriptive `.md` per image. The description is prose on the page that owns the image; the asset is referenced, never narrated in isolation.
4. Obsidian's attachment folder is set to `raw/assets`, so drag-drop in Obsidian and agent writes converge on the same location. Binaries are committed as-is — **never base64**, never inlined into markdown.

Because the embed uses the bare filename (Obsidian resolves `![[name.ext]]` vault-wide), curated pages don't need the `raw/assets/` path prefix in the embed; the `sources:` frontmatter still records provenance with the full path.

### SCHEMA.md (PKM-tuned)

```markdown
# Vault Schema

## Domain
Personal knowledge management for one user. Two intertwined domains:
(1) a research/reference knowledge base (Karpathy LLM-Wiki layers), and
(2) life-admin: tasks, a media-to-consume backlog, and personal memories.
The vault is the single source of truth. Hindsight indexes it; it is never the store.

## Layers
- raw/      Immutable sources. The agent READS but NEVER edits these.
- entities/ concepts/ comparisons/ queries/   Curated, cross-linked knowledge pages.
- actions/ media/ memories/ people/   Life-admin pages (frontmatter `type` is the contract).
- index.md (Home) / SCHEMA.md / log.md   Navigational + structural backbone.

## Conventions
- File names: lowercase, hyphens, no spaces, no dates (dates live in frontmatter).
- Every page starts with YAML frontmatter (see Frontmatter).
- Link with [[wikilinks]]; every knowledge page needs >= 2 outbound links.
- Bump `updated` on every edit. Add every new page to index.md. Append every action to log.md.
- Images: embed inline with ![[asset.ext]] on the owning page AND describe them there.
  Binaries live in raw/assets/. No per-image sidecar files. Never base64.
- Provenance: on pages synthesising 3+ sources, append ^[raw/articles/source.md] to
  paragraphs whose claims trace to one source.

## Frontmatter
[the common + knowledge + life-admin contract above]

## raw/ Frontmatter
---
source_url: https://example.com/article   # if applicable
ingested: YYYY-MM-DD
sha256: <hex digest of the body below the closing --->
---
Compute sha256 over the body only. On re-ingest of the same URL: recompute, compare,
skip if identical, flag drift + update if changed.

## Tag Taxonomy
Add a tag HERE before using it (prevents sprawl). Seed set:
- Knowledge meta: entity, concept, comparison, query, summary, reference, how-to
- Domain (user-specific): embedded-systems, controls, accelerator, software, ai-ml, home
- People/Orgs: person, org, product, model
- Life-admin: task, media, memory, recurring, errand
- Quality: contested, prediction, controversy

## Page Thresholds
- CREATE a page when an entity/concept appears in 2+ sources OR is central to one.
- ADD to an existing page when a source mentions something already covered.
- DON'T create pages for passing mentions or out-of-scope detail.
- SPLIT a page over ~200 lines into sub-topics with cross-links.
- ARCHIVE fully-superseded pages to _archive/ and drop them from index.md.
- Life-admin pages are created on demand (one capture = one action/media/memory page)
  and do NOT need the 2-source threshold.

## Update Policy
On conflict: prefer newer dates; if genuinely contradictory, record both with dates
and sources, set `contradictions:` / `contested: true`, and flag in the lint report.
Never silently overwrite.
```

### index.md — the Home landing page

The index doubles as the Obsidian Home page: a human dashboard (via embedded Bases) **and** the agent's content catalog. Knowledge sections list one wikilink + one-line summary per page; life-admin is shown as live Bases views so it never goes stale.

```markdown
---
title: Home
type: summary
cssclasses: dashboard-full-width
updated: 2026-05-30
---

# 🏠 Hermes Vault — Home

> Source of truth for knowledge + life-admin. Total pages: N | Updated: YYYY-MM-DD
> Agents: read SCHEMA.md, this index, and recent log.md before any operation.

## 📥 Inbox (needs filing)
![[_bases/inbox.base]]

## ✅ Actions
![[_bases/actions.base]]

## 🎬 Media — to consume
![[_bases/media.base]]

## 🧠 Memories
![[_bases/memories.base]]

---

## Knowledge catalog
> One line per page: [[link]] — summary. Alphabetical within section.

### Entities
- [[program-motion-controller]] — central coordinator in the motor-control stack.

### Concepts
- [[ioc-network-architecture]] — RTEMS IOC boot/VLAN/TFTP design notes.

### Comparisons

### Queries

### People
- [[people/jane-doe]] — collaborator on home + controls work.
```

Scaling rule (from llm-wiki): split any knowledge section over 50 entries by first letter/sub-domain; once the catalog passes 200 entries, add `_meta/topic-map.md` and link it here.

### log.md

```markdown
# Vault Log

> Chronological record of all agent actions. Append-only.
> Format: `## [YYYY-MM-DD] action | subject`
> Actions: ingest, create, update, query, lint, archive, delete, reindex
> Rotate when this file exceeds 500 entries OR at year end: rename to log-YYYY.md, start fresh.

## [2026-05-30] create | Vault initialized
- Migrated from documents/ proto-vault
- Structure: raw/{articles,papers,transcripts,assets}, entities/, concepts/, comparisons/,
  queries/, actions/, media/, memories/, people/, _bases/, _archive/, inbox/
```

### Obsidian Bases dashboards (with a Dataview fallback)

> **⚠️ Feasibility caveat — confirm Bases before treating it as load-bearing.** Obsidian **Bases** is a very new first-party feature with an evolving `.base` syntax, and **no skill in this repo documents it** — the in-repo `research/llm-wiki` skill recommends **Dataview** instead (`TABLE … FROM "entities" WHERE …`). The dashboards, Home page, and *all* life-admin surfacing rest on Bases, so before shipping it as the navigation layer: (1) confirm the installed Obsidian version actually ships Bases, and (2) confirm the exact `.base` filter/date syntax below parses (especially the date arithmetic — Open Question #6).
>
> **Concrete fallback / v1 decision.** The **v1 target is Bases** *if it validates on the installed build*; otherwise fall back to **Dataview** (already supported by the llm-wiki skill) — a `dataview` code block per view on the relevant index/Home page, e.g. open actions:
> ````
> ```dataview
> TABLE status, due_date, priority, project
> FROM "actions"
> WHERE status != "done" AND status != "completed" AND status != "cancelled"
> SORT priority ASC, due_date ASC
> ```
> ````
> A second fallback is **status-only Bases filters** (no date arithmetic) with the cron daily-briefing (§11) doing all date math (due/overdue/next-3-days), which it does anyway from frontmatter. Pick one and record it; do not leave the whole dashboard layer gated on the open question.

Bases definition files live in `_bases/` and are embedded by `index.md`. **Critical filter syntax: every `filters:` block MUST be an object keyed by exactly one of `and:` / `or:` / `not:`. A bare YAML list is a parse error; even a single condition is wrapped.** Nest `and`/`or`/`not` objects for compound logic. (See §17 for the Bases-version validation caveat on date arithmetic.)

`_bases/actions.base` — open tasks, then everything:

```yaml
filters:
  and:
    - file.inFolder("actions")
    - file.ext == "md"
properties:
  status:    { displayName: Status }
  due_date:  { displayName: Due }
  priority:  { displayName: Priority }
  project:   { displayName: Project }
  recurrence: { displayName: Repeat }
views:
  - type: table
    name: Open Actions
    filters:
      and:
        - status != "done"
        - status != "completed"
        - status != "cancelled"
    order: [file.name, due_date, priority, status, project, recurrence]
    sort:
      - { property: priority, direction: ASC }
      - { property: due_date, direction: ASC }
  - type: table
    name: Due Soon
    filters:
      and:
        - status != "completed"
        - due_date != ""
        - due_date < now() + "7 days"
    order: [file.name, due_date, priority]
  - type: table
    name: All Actions
    order: [file.name, due_date, priority, status, project]
```

`_bases/media.base` — backlog plus an OR view across the active states:

```yaml
filters:
  and:
    - file.inFolder("media")
    - file.ext == "md"
properties:
  media_type: { displayName: Type }
  creator:    { displayName: Creator }
  priority:   { displayName: Priority }
  status:     { displayName: Status }
  url:        { displayName: URL }
views:
  - type: table
    name: To Consume
    filters:
      and:
        - status == "to_consume"
    order: [file.name, media_type, creator, priority]
    sort: [{ property: priority, direction: ASC }]
  - type: table
    name: In Progress or Done
    filters:
      or:
        - status == "consuming"
        - status == "consumed"
    order: [file.name, media_type, status, creator]
```

`_bases/memories.base`:

```yaml
filters:
  and:
    - file.inFolder("memories")
    - file.ext == "md"
properties:
  people:      { displayName: People }
  location:    { displayName: Location }
  memory_date: { displayName: When }
  tags:        { displayName: Tags }
views:
  - type: table
    name: All Memories
    order: [file.name, people, location, memory_date, created, tags]
    sort: [{ property: memory_date, direction: DESC }]
```

`_bases/inbox.base` — surfaces unfiled captures using a `not:` filter to exclude noise:

```yaml
filters:
  and:
    - file.inFolder("inbox")
    - file.ext == "md"
properties:
  title:   { displayName: Title }
  created: { displayName: Captured }
  tags:    { displayName: Tags }
  source:  { displayName: Via }
views:
  - type: table
    name: Needs Filing
    filters:
      not:
        - file.name == "README"
    order: [file.name, created, source, tags]
    sort: [{ property: created, direction: DESC }]
```

`_bases/home.base` (optional union view embedded near the top of `index.md`) — recent activity across the whole vault:

```yaml
filters:
  and:
    - file.ext == "md"
views:
  - type: table
    name: Recent Captures (7d)
    filters:
      and:
        - file.mtime > now() - "7 days"
        - not:
            - file.inFolder("_archive")
    order: [file.name, type, created, file.mtime, tags]
    sort: [{ property: file.mtime, direction: DESC }]
```

### Provenance, confidence, and contradictions

The vault records *how well it knows what it knows*, so the agent can hedge honestly:

- **Provenance.** `sources:` frontmatter lists the raw files a page synthesises. On pages that fuse 3+ sources, claims carry inline footnote markers `^[raw/articles/source.md]` so a reader can trace a specific assertion to its origin.
- **Confidence.** `confidence: high|medium|low` (default unset ⇒ medium). Lint flags `low` and any single-source page with no `confidence`, prompting corroboration or demotion.
- **Contradictions.** When two sources genuinely conflict, the agent records both with dates and sources, sets `contested: true` and `contradictions: [other-slug]`, and never silently overwrites. Lint surfaces every contested page and same-topic pages stating different facts.

---

## 7. Capture / Ingestion

Ingest is the heart of capture. It runs the same whether triggered by a Slack DM, an MCP call, or a cron job. **Pull before write, commit+push after.** The persist step is an explicit numbered action (step 6) — never a parenthetical — to defeat the acknowledge-without-persist bug.

### The INGEST operation (step-by-step)

```
0. ORIENT (once per session). Read SCHEMA.md, index.md, and the last ~30 lines of log.md.
   For 100+ page vaults, also search before creating anything new.
   git: vault-pull (pull --rebase --autostash) from the vault remote so we write onto current state.

1. CAPTURE RAW.
   - URL        -> Exa search to find / Firecrawl extract to markdown -> raw/articles/<slug>.md
   - PDF/arxiv  -> Firecrawl/extract to markdown -> raw/papers/<slug>.md AND keep the original
                  binary PDF alongside it at raw/papers/<slug>.pdf (papers keep their source PDF in
                  raw/papers/, not raw/assets/; assets/ is for embedded images/diagrams)
   - transcript -> raw/transcripts/<slug>.md
   - audio/voice-> STT (config stt.provider, local whisper) transcribes -> raw/transcripts/<slug>.md,
                  then curate the transcript as normal (the raw audio file is incidental, not stored as a vault asset)
   - image      -> raw/assets/<slug>-<shorthash>.<ext>  (binary, as-is, never base64)
   - paste      -> the right raw/ subdir by content type
   Write raw frontmatter: source_url (if any), ingested: <date>,
   sha256: <digest of body>.  Skip step 1 write if an identical sha256 already exists.

2. CHECK EXISTING. Search index.md + grep the vault for every entity/concept named in the
   source. Decide create-vs-update per the Page Thresholds. This is what makes a wiki instead
   of a pile of duplicates.

3. CURATE & CROSS-LINK.
   - Knowledge: create/update entities/concepts/comparisons pages. Each new/updated page gets
     >= 2 [[wikilinks]]; verify back-links. Embed any image with ![[asset.ext]] AND describe it
     on the page. Set confidence for opinion-heavy/single-source claims. Add ^[raw/...] provenance
     on 3+ source pages. Only taxonomy tags.
   - Life-admin: if the capture is a task/media/memory, write ONE page to actions/ | media/ |
     memories/ with the type-specific frontmatter (status, due_date, priority, media_type,
     people, location, memory_date, recurrence as applicable). Link people to people/ pages.
   - Truly ambiguous -> inbox/ with type: inbox for later filing.

4. UPDATE NAVIGATION.
   - Add new knowledge pages to index.md under the right section (alphabetical); bump
     "Total pages" + "Updated". Life-admin needs no index edit — Bases surface it live.
   - Append to log.md: `## [YYYY-MM-DD] ingest | <Source Title>` listing every file touched.

5. HAND OFF TO HINDSIGHT. Call hindsight_retain for the CURATED page (page path + title +
   summary + key claims) — NOT the raw conversation. Attach the vault-relative path to every
   fact: as reference=<path> if the installed client supports it, ELSE prefix the fact text
   with a "SOURCE: <path>" sentinel line and add the path to tags (see §9 prerequisite). Recall
   then returns vault page pointers, not chatter.
5a. VALIDATE THE RETAIN LANDED (mirror the vault-write "did it land?" discipline; auto_retain is
    OFF so this call is the ONLY thing indexing the page). Probe with hindsight_recall for a
    distinctive phrase from the page (or `hindsight-embed -p hermes db query`) and confirm the
    page path comes back. If it does not, flag the page for the nightly reindex rather than
    silently assuming success. (Index is rebuildable; see §9 — vault canonical => index disposable.)

6. SYNC OUT. vault-commit "<subject>": git add -A; commit `agent: ingest <subject>` FIRST (clean
   tree); then pull --rebase onto any Obsidian push that landed during the batch; then push to the
   vault remote. Conflicts are rare (raw/ is immutable, one-file-per-topic); on a rebase collision
   the wrapper aborts the rebase, does NOT push, and surfaces the path over Slack (never clobbers).

7. REPORT. Reply in Slack/MCP listing files created/updated AND clickable retrieval links:
   obsidian://open?vault=pkm-vault&file=<url-encoded path>  plus the vault-relative path /
   wikilink. (Slack file-upload of the attachment is the fallback.)
```

### Content types and auto-categorization rules

The router/classifier maps each capture to exactly one `type`:

| Signal in the capture | type | Folder |
|---|---|---|
| Question directed at the agent | (no file) | answer inline; file to `queries/` only if non-trivial |
| Source to remember/synthesise (URL, PDF, pasted reference, how-to, explainer) | `concept` (or `entity` if it is one named thing) | `concepts/` / `entities/` |
| Audio / voice memo upload (Slack audio file) | (transcribe first, then route by content) | STT (`stt.provider: local` whisper) → `raw/transcripts/<slug>.md`, then curate as normal |
| A named person/org/product/model/device | `entity` | `entities/` (people → `people/`) |
| Explicit task / "remind me" / "I need to…" / `#project` task | `action` | `actions/` |
| Book/film/TV/podcast/article/video/music to consume later | `media` | `media/` |
| Personal/emotional moment, family, photo of people/places, milestone, holiday | `memory` | `memories/` |
| Side-by-side "X vs Y" analysis | `comparison` | `comparisons/` |
| Genuinely ambiguous | `inbox` | `inbox/` |

Disambiguation rules:
- Personal/emotional ⇒ `memory`, not `concept`, even if informational.
- For a YouTube/music URL captured for later, extract the real title from the page/URL for both `title` and the displayed name; never "YouTube Video".
- `#projectslug` in the message forces `project: "[[projectslug]]"` on an action and links it to that project page, which lives in `entities/` (a named project) or `concepts/` (a body of work) — there is no `projects/` folder. The legacy 2ndBrain `Reference` category maps to `concepts/`.
- An image alone is **not** a category — it attaches to whichever page owns it (entity/concept/memory), embedded and described there.

### Slack / MCP / attachments

- **Slack:** pasted URLs, typed thoughts/TODOs, and inline file/photo/PDF uploads all flow through the same INGEST operation; the uploaded binary is pulled from Slack into `raw/assets/`.
- **MCP:** text captures and path-referenced files; no Slack-style upload, so MCP capture leans on the calling Claude passing content or a path.
- **Attachments:** binaries land immutably in `raw/assets/` under a descriptive content-addressed slug and are embedded by the curated page; the runtime `image_cache/` copy is incidental and gitignored.

### Updated `SOUL.md` persona

The current `SOUL.md` encodes the retired model — it literally states *"the markdown files are just convenient snapshots — the DB is the truth"* and instructs the agent to persist into Hindsight via `memory(action='add')` as the store. The replacement persona below makes the **vault canonical**, makes Hindsight a derived index, and bakes in `obsidian://` retrieval and a concise tone. It supersedes the existing file in full.

```markdown
# Hermes PKM Agent Persona

You are a Personal Knowledge Management assistant — a second brain for one user
(Giles, Europe/London). You capture knowledge into a canonical Obsidian vault,
retrieve it with structural + semantic search, and always point the user back to
the real note in their own Obsidian.

## Source of truth
- The **Obsidian vault** (markdown files + binary assets in `raw/assets/`) is the
  ONLY canonical store. It is a git repo, two-way synced with the user's workstation.
- **Hindsight is a rebuildable index over the vault**, not a store. If it drifts,
  it gets reindexed from the vault.
- The Hermes session DB (`state.db`) is **transient working memory** — never the
  knowledge base. Do NOT treat ingested content as "saved" because it is in a
  session or in Hindsight; it is saved only when it is a committed vault file.

## Capturing content (throw-it-and-forget)
1. Detect type: URL, markdown note, code, idea, quote, image, PDF, TODO/Action,
   media-to-consume, memory.
2. `vault-pull` the vault first (pull --rebase).
3. Immutable sources (uploaded articles/papers/transcripts/images) go to `raw/`
   (images to `raw/assets/`). Write the curated, cross-linked page in the right
   layer (entities / concepts / comparisons / queries) per SCHEMA.md.
4. Life-admin items (Actions/TODOs, media backlog, memories) are wiki pages with a
   frontmatter `type:` — never a rival folder tree. Set due/recurrence/priority on
   Actions from natural language.
5. Embed images inline with Obsidian wiki-embeds; the curated page describes AND
   embeds the asset. Never store base64. Never write a separate descriptive sidecar.
6. Auto-tag and cross-link. Never ask the user to file or tag.
7. hindsight_retain the page, attaching its vault path (reference=<path> if supported, else a
   `SOURCE: <path>` sentinel line + path tag); probe with hindsight_recall that the page path
   comes back (auto_retain is off, so this is the only thing indexing the page). Append to
   `log.md`; then `vault-commit`.
8. Confirm in 1–2 lines: what it is, where it landed, the tags applied.

## Retrieving content
1. Navigate structurally first (folders, `index.md`, wikilinks, Bases views), then
   use Hindsight semantic recall over CURATED pages to find by meaning.
2. Answer concisely from the vault, then ALWAYS offer the source:
   `obsidian://open?vault=pkm-vault&file=<url-encoded vault-relative path>`
   plus the plain vault-relative path and a `[[wikilink]]`.
3. Slack: render as mrkdwn `<url|title>`. MCP: markdown `[title](url)` + raw path +
   wikilink (the host may not make the custom scheme clickable).
4. Offer a Slack file upload ONLY if the user asks or clearly can't reach Obsidian.

## Proactive summaries (cron, Europe/London)
- Daily 07:00 and weekly Mon 07:00 to the user's Slack DM, composed FROM THE VAULT:
  due/overdue Actions, deadlines in the next 3 days, recent ingests, media-backlog
  nudges, emerging themes, review-flagged items. Use wikilinks as handles.

## Tone
- Concise. Acknowledge captures in 1–2 lines. Give retrieval results with their
  source links and nothing extra. You are an efficient, reliable tool, not a
  conversationalist. Prefer clean state — no cruft, no commented-out leftovers.

## Timezone: Europe/London (GMT/BST)
```

---

## 8. Retrieval

**The point of retrieval is to get the user back to the real note in their own Obsidian, not just to read an answer in chat.** On every retrieval the agent replies with (a) a concise answer synthesised from the vault, **plus** (b) one or more clickable `obsidian://` deep links that open the exact source note or attachment in Obsidian on the user's device. Vault-relative paths and wikilinks are offered alongside as fallbacks. Slack file-upload of the attachment is a *last-resort* fallback, not the default.

### The QUERY operation

The agent reaches for retrieval mechanisms in cost order — structure first, semantics last — and always hands back a vault path regardless of which mechanism found the answer:

```
query
  → read index.md (cheap, authoritative)            # structural orientation
  → known term or acronym?  → search_files            # lexical (FTS/grep over the vault)
  → known starting page?    → follow [[wikilinks]]    # graph navigation
  → phrasing-independent, "find me the page about…"   # semantic
        → hindsight_recall  →  returns vault page references
  → "connect / synthesize across pages"
        → hindsight_reflect
  → COMPOSE answer from the vault page(s)
  → RETURN answer + obsidian:// deep link(s) + vault-relative path(s) + [[wikilink]](s)
```

Structural navigation answers *"open the page I know exists"*; semantic recall (§9) answers *"find the page I can't name."* Hindsight earns its keep precisely when the user's words don't match the vault's words.

### `obsidian://` deep links (the primary retrieval handle)

#### Link format

```
obsidian://open?vault=<VAULT_NAME>&file=<URL_ENCODED_VAULT_RELATIVE_PATH>
```

- **`vault=<VAULT_NAME>`** — the registered Obsidian vault name on the user's workstation, **not** a filesystem path. This is a fixed configuration value (`pkm-vault`, matching the vault repo / Obsidian vault folder name) stored once as `OBSIDIAN_VAULT_NAME`, so every reply uses the identical string Obsidian registered. If the user renames the vault in Obsidian, this one value is updated; nothing else changes.
- **`file=<path>`** — the **vault-relative** path (no leading slash, including the `.md` extension for notes; the file extension is **required** for non-markdown attachments), **URL-encoded** in full (path separators included).

#### Exact path encoding

Percent-encode space → `%20`, `/` → `%2F`, `#` → `%23`, `&` → `%26`, `?` → `%3F`, and other reserved characters per RFC 3986. The agent constructs the link by URL-encoding the raw vault-relative path string in full — the form Obsidian's URI handler accepts.

| Vault-relative path | Encoded `file=` value |
|---------------------|------------------------|
| `entities/exa-search.md` | `entities%2Fexa-search.md` |
| `raw/papers/cap-theorem-2002.pdf` | `raw%2Fpapers%2Fcap-theorem-2002.pdf` |
| `raw/assets/motor-control-diagram-e4a408.png` | `raw%2Fassets%2Fmotor-control-diagram-e4a408.png` |
| `concepts/distributed systems.md` | `concepts%2Fdistributed%20systems.md` |

Full example link (single line):

```
obsidian://open?vault=pkm-vault&file=raw%2Fassets%2Fmotor-control-diagram-e4a408.png
```

#### How links render per interface

| Interface | Rendering | Notes |
|-----------|-----------|-------|
| **Slack** | `mrkdwn` link: `<obsidian://open?vault=pkm-vault&file=entities%2Fexa-search.md\|Open in Obsidian: exa-search>` renders as clickable **Open in Obsidian: exa-search**. | Slack shows a one-time confirmation on the non-http scheme; that is expected. If the workspace strips custom schemes, fall back to posting the raw `obsidian://` URI as plain text plus the vault-relative path. Keep link text short and human (the note title). |
| **MCP (Claude Code / claude.ai)** | Standard markdown link `[Open in Obsidian: exa-search](obsidian://open?vault=pkm-vault&file=entities%2Fexa-search.md)` **and** the raw vault-relative path **and** a `[[wikilink]]`. | The host may not make a custom-scheme link clickable, so always include the **plain path** and the wikilink so the user (or the calling Claude) can navigate regardless. |

Always include the **vault-relative path and `[[wikilink]]`** in both interfaces — they are scheme-independent, paste-able, and survive copy/paste into Obsidian directly.

### Returning originals (and when to upload)

The `obsidian://` link is always primary. Add a Slack file upload **only** when the user cannot reach Obsidian for that artifact:

- The user explicitly asks to *see it here* / *send the file* / *upload it*.
- The artifact is a **binary attachment** (image/PDF) **and** the user is on a device where Obsidian may not be installed/synced — offer it as a one-line "want me to upload it here?" rather than dumping the file by default.
- Never upload large binaries pre-emptively (cost/clutter, and the user prefers clean state). For markdown notes, prefer the link + an inline excerpt over uploading the `.md`.

### Worked example (Slack reply)

> **User:** what was that motor control diagram I saved, and the CAP theorem paper?
>
> **Agent:**
> Two hits from the vault:
>
> **1. Motor Control System Diagram** — hand-drawn architecture: PMC (Program Motion Controller) coordinates, DCM drives the hardware interface, CS Motor / CS Demands feed in, Motor Rail/API is the physical link. Several components are checked off as done. Filed under entities, image embedded inline.
> · Open note → **<obsidian://open?vault=pkm-vault&file=entities%2Fprogram-motion-controller.md|entities/program-motion-controller>**
> · Open image → **<obsidian://open?vault=pkm-vault&file=raw%2Fassets%2Fmotor-control-diagram-e4a408.png|raw/assets/motor-control-diagram-e4a408.png>**
> · Wikilink: `[[program-motion-controller]]`
>
> **2. CAP Theorem (Brewer/Gilbert-Lynch)** — paper on consistency/availability/partition-tolerance trade-offs; linked from your `concepts/distributed-systems` page.
> · Open note → **<obsidian://open?vault=pkm-vault&file=entities%2Fcap-theorem.md|entities/cap-theorem>**
> · Open PDF → **<obsidian://open?vault=pkm-vault&file=raw%2Fpapers%2Fcap-theorem-2002.pdf|raw/papers/cap-theorem-2002.pdf>**
> · Wikilink: `[[cap-theorem]]`
>
> Want either file uploaded here instead?

The MCP variant swaps the `mrkdwn` `<url|label>` syntax for markdown `[label](url)` links and keeps the bare paths and wikilinks verbatim.

---

## 9. Semantic Index (Hindsight)

Hindsight is the system's **semantic recall layer**: a rebuildable index that lets the agent find vault pages by *meaning* rather than by filename or wikilink. It is open-source agent memory by Vectorize.io (`github.com/vectorize-io/hindsight`), wired into Hermes as the `memory.provider: hindsight` plugin running in `local_embedded` mode.

> **The single most important rule for this layer:** Hindsight indexes **CURATED vault content** — filed pages and their salient facts — each tagged with the vault-relative page path. It does NOT index raw conversation. Recall therefore returns *vault page references*, not chat snippets. The vault is canonical; the Hindsight index is disposable and rebuildable from the vault at any time. This retires the as-built `SOUL.md` model that conflates "Hindsight DB" with the SQLite session store and asserts "the DB is the truth."

> **⚠️ Hard prerequisite gating this entire section (formerly Open Question #4).** The page-pointer mechanism below depends on each retained fact carrying its source vault path so that recall can return *a path*, not just text. The as-built docs attest the tool only as `hindsight_retain` = "explicitly store a fact with optional tags" (`productivity/.../hindsight-embeddings.md:83`) and expose `hindsight_retain` / `hindsight_recall` / `hindsight_reflect` with **no documented `reference` argument** and **no attested Python `Hindsight` class** — only the `hindsight-embed` CLI and the in-agent `hindsight_*` tools are confirmed. **Before building on it, verify the installed `hindsight_retain` / client signature.** The spec therefore specifies the path-pointer in a way that does **not** require an unverified `reference=` parameter:
>
> - **Path carried *inside the retained text* as a sentinel (the portable, no-new-API design — use this unless `reference=` is verified).** Every retained fact is prefixed with a machine-parseable line `SOURCE: <vault-relative-path>` (e.g. `SOURCE: entities/program-motion-controller.md`). Recall results then contain that token; the chat layer parses the first `SOURCE:` line back into the `obsidian://` link and vault path. This rides on the *attested* "store a fact with optional tags" surface (the path can also be duplicated into `tags`), needs no `reference` parameter, and survives a Hindsight version that lacks one.
> - **`reference=<path>` (the cleaner design, *only if verified to exist*).** If the installed client/tool accepts a first-class `reference` field that recall echoes back, use it instead of (or in addition to) the sentinel — it is tidier and makes per-page `forget`/upsert natural. Until verified, treat it as optional sugar layered over the sentinel, not a dependency.
>
> Everywhere below that shows `reference=<path>` is shorthand for "attach the path as a first-class reference **if supported, else as the `SOURCE:` sentinel + tag." The retrieval contract ("recall returns a vault page path") holds either way.

### Architecture: four memory networks

Hindsight organizes what it retains into four cooperating **memory networks**; TEMPR retrieval draws from all of them.

| Network | What it holds | PKM mapping (what we retain into it) |
|---|---|---|
| **World facts** | Stable, objective facts | Salient declarative facts extracted from curated `entities/`, `concepts/`, `comparisons/` pages, each carrying the source page path as a reference |
| **Bank / experience** | Episodic, time-stamped events | Ingest events and filed query results: "on YYYY-MM-DD a source on X was filed to `entities/x.md`"; surfaced from `queries/` and `log.md` |
| **Opinion** | Subjective stances, preferences | `confidence`/`contested` page positions, the user's stated preferences captured in `USER.md`, verdicts on `comparisons/` pages |
| **Observation** | Raw observations awaiting consolidation | The short-lived staging area for facts seen in conversation **before** they are filed. Kept thin by design (`auto_retain: false`) so transient chatter does not harden into recallable memory |

The `bank_id` for this deployment is **`hermes`** — a single memory bank spanning all four networks.

#### Operations: retain / recall / reflect

Hindsight exposes three primitives as Hermes tools (`hindsight_retain`, `hindsight_recall`, `hindsight_reflect`):

- **retain(text, tags[, reference])** — extract facts from `text` via Gemini, route each fact to the appropriate network, embed it, and persist it. **We always attach the vault-relative page path** (e.g. `entities/program-motion-controller.md`) — as a first-class `reference` if the installed client supports it, otherwise as a `SOURCE: <path>` sentinel line prefixed onto `text` plus a path tag (see the hard prerequisite above). This is the hook that makes recall return page references; the attested surface is "store a fact with optional tags", so the sentinel path is the no-new-API guarantee.
- **recall(query, budget)** — embed the query and run **TEMPR** to return the most relevant facts; the chat layer reads each fact's `reference` (or parses its `SOURCE:` line) back to the vault page path.
- **reflect(query)** — LLM-powered synthesis *across* multiple retained facts to surface non-obvious connections. Used for cron summaries and "what have I been learning?" questions, not routine lookup.

#### TEMPR retrieval (parallel multi-strategy)

A single `recall` does **not** rely on vector similarity alone. Hindsight's **TEMPR** retrieval runs four searches **in parallel** and fuses the results:

```
recall(query)
  ├── semantic     →  k-NN over embeddings (meaning match: "auth" ≈ "login system")
  ├── BM25         →  lexical/keyword match (exact terms, acronyms like "PMC", "DCM")
  ├── entity-graph →  walk relationships between retained entities (find linked facts)
  └── temporal     →  recency / time-window match (what was filed recently, when)
        ↓
   fuse + rank  →  top facts, each with its vault page `reference`
```

BM25 catches domain acronyms that embeddings blur together; the entity-graph leg mirrors the vault's own `[[wikilinks]]` at the index level; the temporal leg powers "what did I ingest yesterday?" in the daily summary. The four legs run on every recall — there is no manual escalation step (which retires the as-built "FTS5 first, escalate to semantic" flow).

### Integration as the Hermes `local_embedded` provider

Hindsight runs **entirely on the VPS** with no cloud Hindsight account. Gemini is used for both fact extraction and embeddings; a **local PostgreSQL daemon** holds the index.

`~/.hermes/hindsight/config.json` (tuned for vault-mirroring):

```json
{
  "mode": "local_embedded",
  "bank_id": "hermes",
  "llm_provider": "gemini",
  "llm_model": "gemini-2.5-flash",
  "auto_recall": true,
  "auto_retain": false,
  "memory_mode": "hybrid",
  "recall_budget": "mid",
  "recall_prefetch_method": "recall",
  "recall_max_tokens": 4096,
  "retain_async": true,
  "retain_context": "facts already filed into the Obsidian vault by Hermes"
}
```

The substantive changes from the as-built file are **`auto_retain: false`** (was `true`), the rewritten **`retain_context`**, and the **extraction model** moved off the experimental `gemini-2.0-flash-exp` (which Google deprecates) onto the supported **`gemini-2.5-flash`** that the in-repo Hindsight reference skill already uses; everything else (mode, bank, Gemini provider, async retain, mid budget, 4096 recall tokens) is retained. The now-inert `retain_every_n_turns` is dropped from the target file (see the tuning table below: it only governs auto-harvest, which is off under `auto_retain: false`).

**Two models, two roles — and only one is `llm_model`.**

- **`llm_model` controls fact *extraction* only** (the Gemini chat model that distils facts from page text). The target is `gemini-2.5-flash`.
- **Embeddings are a *separate* model.** Hindsight's `local_embedded`/Gemini path embeds with **`text-embedding-004`**, producing **3,072-dimensional** vectors (per the in-repo `hermes-pkm-embeddings` skill). This is the model the free-tier sizing below is anchored to: Gemini's free tier covers ~60,000 embeddings/month for `text-embedding-004`. (One in-repo curl sample references the older `embedding-001`; standardise on `text-embedding-004` / 3072-dim, which is what the embeddings skill documents.) Whether the embedding model is independently configurable in `config.json` (vs fixed by the Hindsight build for `llm_provider: gemini`) is **not** attested in the installed docs — treat `text-embedding-004` / 3072-dim as the expected default and **verify it against the installed Hindsight version** (Open Question #5). `llm_model` does **not** select the embedding model.

**Daemon mechanics.**

- **Auto-start / auto-stop:** On the first `hindsight_recall`/`hindsight_retain` call, the daemon starts a PostgreSQL **subprocess** (not a system service). It **auto-stops after ~5 min idle** (`HINDSIGHT_IDLE_TIMEOUT`, default 300s). No `systemctl` unit; the Hermes gateway owns its lifecycle.
- **Root-safe:** `local_embedded` has built-in root handling — the "PostgreSQL cannot run as root" worry is a non-issue on the isolated VPS.
- **Data location:** index data lives under `~/.hermes/hindsight/db/`; daemon binaries under `~/.hermes/hindsight/daemon/`. **This is index data, not a store of record — it is `.gitignore`d and never committed to any repo.**
- **API key:** Gemini key via `HINDSIGHT_LLM_API_KEY` in `~/.hermes/.env` (Hermes also reads `GEMINI_API_KEY`/`GOOGLE_API_KEY`). This is the **Gemini free-tier** key, entirely separate from the Anthropic pay-as-you-go API key — Hindsight never touches Anthropic.

**Daemon management commands:**

```bash
# Binary name `hindsight-embed` and these subcommands are attested only in agent-authored skill
# docs, not upstream — confirm `hindsight-embed --help` / `... ui start` on the INSTALLED package.
tail -f ~/.hermes/logs/hindsight-embed.log     # tail daemon logs
hindsight-embed -p hermes db query             # inspect the index (sanity-check page facts)
killall hindsight-embed                         # manual stop (auto-restarts on next memory call)
hermes gateway restart                          # after config.json edits, recycle the gateway
hindsight-embed -p hermes ui start             # optional local_embedded web UI — port http://localhost:8080
                                                #   (8080 is the local_embedded UI; verify per installed
                                                #    version — Docker uses 9999 UI / 8888 API, see below)
```

> Tuning tooling and daemon command surface are taken from the in-repo references; see §17 for the items to verify against the installed Hindsight version (exact client method names, UI port, idle-timeout env var).

### Tuning: mirror the wiki, not the transcript

| Setting | As-built | This spec | Rationale |
|---|---|---|---|
| `auto_retain` | `true` | **`false`** | Don't auto-harvest raw conversation. The index gains content **only** via deliberate `hindsight_retain` calls at file-time (per-ingest) and via the reindex job. The vault is the gate; nothing enters the index that isn't first a curated page. |
| `auto_recall` | `true` | `true` (kept) | Pre-fetching relevant **page facts** into context before each turn surfaces the right pages, not chatter. |
| `memory_mode` | `hybrid` | `hybrid` (kept) | Auto-recall injected into context **and** the `hindsight_*` tools exposed for explicit retain/recall/reflect. |
| `recall_budget` | `mid` | `mid` (kept) | Balanced cost/recall for a personal-scale vault. Raise to `high` only if recall misses relevant pages past a few thousand pages. |
| `recall_max_tokens` | `4096` | `4096` (kept) | Caps recalled material injected per turn. |
| `retain_async` | `true` | `true` (kept) | Retain/embed off the critical path so chat stays responsive. |
| `retain_every_n_turns` | `1` | **removed (no-op)** | Governs *auto-harvest* cadence only; with `auto_retain: false` nothing auto-harvests, so this key is inert. Dropped from the target `config.json` to avoid implying auto-harvest still runs. |
| `retain_context` | "conversation between Hermes Agent and the User" | **"facts already filed into the Obsidian vault by Hermes"** | Steers Gemini's extraction toward *page* facts and away from conversational framing. |

> **Net effect:** the contents of the Hindsight index are a function of the vault, not of the chat history. Read back the index (`hindsight-embed -p hermes db query`) and every fact traces to a vault page path. Reindex-from-vault is therefore guaranteed to reproduce the index.

> **Validation discipline (because `auto_retain` is off).** Per-ingest `hindsight_retain` is now the *primary* indexing path, and a skipped call leaves the page invisible to recall until the nightly reindex. So the ingest flow (§7 step 5a) treats retain like a vault write: after retaining, **probe** with `hindsight_recall` (or `hindsight-embed -p hermes db query`) that the page path is present; if absent, flag the page for the nightly catch-up. This applies the same "acknowledge-without-persist" guard the spec mandates for file writes to the index write.

### Retain example — page references, not chatter

For the curated page `entities/program-motion-controller.md`, Hindsight retains (illustrative, **every fact carrying the same path** — as a `reference` if supported, else via the `SOURCE:` sentinel + `tags`):

```text
path = "entities/program-motion-controller.md"
  (attached as reference=<path> if the client supports it; otherwise each fact text
   begins with a sentinel line "SOURCE: entities/program-motion-controller.md", and
   the path is also added to tags)
 ├─ world fact: "PMC = Program Motion Controller, central coordinator of the motor-control stack"
 ├─ world fact: "PMC drives the DCM (Drive Control Module) hardware interface"
 ├─ world fact: "PMC consumes CS Demands from the control system"
 └─ (entity-graph edges: PMC—DCM, PMC—IOC network architecture)
```

A recall returns the **page path** (echoed `reference` or parsed `SOURCE:` line), which the chat layer renders as a clickable `obsidian://` deep link plus the vault-relative path — a pointer into the canonical vault, not a regurgitated chat snippet.

### The reindex-from-vault job (key deliverable)

This is the mechanism that makes "vault canonical, index disposable" real. The job walks **every** curated markdown page in the vault and retains its salient facts into Hindsight, keyed by page path, **skipping pages unchanged since the last run**.

**Idempotency contract.**

- One Hindsight reference == one vault-relative page path (e.g. `concepts/cap-theorem.md`).
- A **per-page content hash** over the page **body** (everything after the closing frontmatter `---`) decides whether to reprocess — reusing the wiki's `sha256`-of-body convention. The hash for each curated page is tracked in an index-side manifest **outside** the vault (`~/.hermes/hindsight/reindex-manifest.json`, `.gitignore`d), so reindex never churns curated pages' `updated:` dates.

**Algorithm.**

> **⚠️ Illustrative pending API verification.** The class/method names below (`Hindsight(...)`, `hs.forget`, `hs.forget_bank`, `hs.retain(reference=…)`) are **not attested** in the installed deployment — the only confirmed local interfaces are the **`hindsight-embed` CLI** and the in-agent `hindsight_*` tools (the pip package is `hindsight-client`, but its Python class name, constructor, and `forget`/`forget_bank`/upsert-by-reference semantics are unverified — Open Question #4/#5). **Do not ship this verbatim.** Two paths: (A) the **subprocess-over-CLI** version (below the Python sketch) drives only the attested `hindsight-embed -p hermes` surface and is the safe concrete implementation; (B) if you verify the Python client's real symbols, the Python sketch is the cleaner port. The body-hash idempotency, `SOURCE:`-sentinel path attachment, and prune-on-delete logic are identical between them. The Python version is kept as the design reference:

```python
#!/usr/bin/env python3
# ~/.hermes/hindsight/reindex_from_vault.py  — ILLUSTRATIVE (verify client symbols first)
# Rebuilds / refreshes the Hindsight index from the canonical Obsidian vault.
# Idempotent: unchanged pages (by body sha256) are skipped.
#
# WARNING: `Hindsight`, `.forget`, `.forget_bank`, and retain(reference=) are NOT verified
# against the installed `hindsight-client`. Confirm the real symbols (Open Q#4) or use the
# CLI/subprocess variant below, which only touches the attested `hindsight-embed` surface.

import hashlib, json, os, re
from pathlib import Path
from datetime import datetime, timezone
# from hindsight_client import Hindsight   # <-- UNVERIFIED symbol; confirm before importing

VAULT     = Path(os.environ["PKM_VAULT"])           # /opt/pkm-vault
MANIFEST  = Path.home() / ".hermes/hindsight/reindex-manifest.json"
BANK      = "hermes"

# Only curated, fact-bearing pages are indexed. raw/ is immutable source bytes;
# navigational/meta files are structure, not facts; underscore dirs are excluded.
INDEXED_DIRS = ("entities", "concepts", "comparisons", "queries")
SKIP_FILES   = {"SCHEMA.md", "index.md", "log.md"}   # index.md IS the Home landing page; there is no separate Home.md

def _now(): return datetime.now(timezone.utc).isoformat()

def body_hash(md: str) -> str:
    body = re.sub(r"^---\n.*?\n---\n", "", md, count=1, flags=re.DOTALL)
    return hashlib.sha256(body.encode("utf-8")).hexdigest()

def page_type(md: str) -> str:
    m = re.search(r"^type:\s*(\S+)", md, flags=re.MULTILINE)
    return m.group(1) if m else "page"

def retain_text(rel: str, md: str) -> str:
    # Portable path-pointer: prefix a parseable SOURCE: sentinel so recall results
    # carry the vault path even if the client has no first-class `reference` field.
    return f"SOURCE: {rel}\n\n{md}"

def main(full_rebuild: bool):
    hs = Hindsight(mode="local_embedded", bank_id=BANK)   # <-- UNVERIFIED constructor
    manifest = {} if full_rebuild else json.loads(MANIFEST.read_text() or "{}")
    if full_rebuild:
        hs.forget_bank(BANK)          # index lost / cold start: wipe and re-derive

    seen, changed, skipped = set(), 0, 0
    for d in INDEXED_DIRS:
        for path in (VAULT / d).rglob("*.md"):
            rel = str(path.relative_to(VAULT))      # the stable page path / pointer
            if path.name in SKIP_FILES:
                continue
            seen.add(rel)
            md  = path.read_text(encoding="utf-8")
            h   = body_hash(md)
            if manifest.get(rel, {}).get("sha256") == h and not full_rebuild:
                skipped += 1
                continue                            # unchanged -> skip (idempotent)

            hs.forget(reference=rel)                # drop stale facts (if upsert-by-ref unsupported)
            hs.retain(                              # re-extract & re-embed via Gemini
                text=retain_text(rel, md),          # SOURCE: sentinel carries the path
                tags=[page_type(md), rel],          # path also tagged for recall filtering
                # reference=rel,                    # add ONLY if verified to exist
                retain_context="a curated page filed in the Obsidian vault",
            )
            manifest[rel] = {"sha256": h, "retained_at": _now()}
            changed += 1

    for gone in set(manifest) - seen:               # prune deleted pages
        hs.forget(reference=gone)
        del manifest[gone]

    MANIFEST.write_text(json.dumps(manifest, indent=2))
    print(f"reindex: {changed} updated, {skipped} unchanged, {len(seen)} live pages")
```

**Concrete CLI/subprocess variant (attested surface — preferred until the Python client is verified).** Drives only `hindsight-embed -p hermes`, the one confirmed `local_embedded` interface. The exact `retain`/`forget`/`query` subcommand spelling must still be confirmed with `hindsight-embed -p hermes --help` (Open Question #5), but the shape is:

```python
import subprocess
BANK_ARGS = ["hindsight-embed", "-p", "hermes"]   # attested CLI; confirm subcommands via --help

def hs_retain(rel: str, md: str, ptype: str):
    # Path travels as the SOURCE: sentinel inside the fact text (no `reference` flag needed).
    subprocess.run(BANK_ARGS + ["retain", "--text", f"SOURCE: {rel}\n\n{md}",
                                "--tags", f"{ptype},{rel}"], check=True)

def hs_forget(rel: str):
    # If the CLI cannot forget-by-path, rely on a --full-rebuild wipe instead (see below).
    subprocess.run(BANK_ARGS + ["forget", "--match", rel], check=False)
```

If the CLI exposes no per-page `forget`, fall back to: incremental runs *add* (duplicate facts are deduped by Hindsight or tolerated), and **`--full-rebuild` is the authoritative reset** — wipe the bank (`hindsight-embed -p hermes db reset` / equivalent, confirm name) and re-retain every live page. Because the vault is canonical, a periodic full rebuild always converges the index regardless of upsert support.

Properties: **idempotent** (no changes ⇒ zero LLM/embedding work); **self-healing on index loss** (run `--full-rebuild` to wipe and deterministically re-derive from the canonical vault — nothing is lost); **deletion-aware** (stale facts pruned where per-page forget exists, otherwise cleared by full rebuild); **scoped to curated knowledge** (`raw/` and underscore/structure files excluded); **client-agnostic** (path carried in-band via the `SOURCE:` sentinel, so it works without a verified `reference` parameter).

**Scheduling & triggering — three paths:**

1. **Per-ingest, incremental (primary).** During an ingest the agent calls `hindsight_retain` for exactly the pages it touched, attaching the page path (first-class `reference` if supported, else the `SOURCE:` sentinel + tag) and probing that it landed (§7 step 5a) — vault write and index update happen together. No separate job for the common case.
2. **Nightly catch-up (safety net).** A Hermes cron job runs the incremental reindex to pick up **out-of-band** edits — pages the user wrote/edited directly in Obsidian and pushed via the Obsidian Git plugin, which the VPS agent never saw. Runs right after the daily pull, before the 07:00 summary:

   ```bash
   hermes cron create "30 6 * * *" \
     "Pull the vault repo, then run reindex_from_vault.py (incremental) to sync \
      the Hindsight index with any pages edited directly in Obsidian overnight." \
     --name "hindsight-reindex" \
     --model '{"model": "gemini-2.5-flash", "provider": "gemini"}' \
     --deliver "slack:D0B61LKA3NV"
   ```

   **Cron CLI form (verified against the as-built `hermes-pkm-ops` skill):** schedule and prompt are **positional** (`hermes cron create "<expr>" "<prompt>" …`), with `--name`/`--model`/`--deliver` flags *after*. The `--model` value is a **JSON object** `'{"model": …, "provider": …}'`, not a slash-string. This job must set an explicit Gemini model — a job left on the default would inherit a Claude model name with the Gemini provider and fail with `HTTP 404` (the documented cron pitfall). Timezone is taken from the global `timezone: Europe/London` in `config.yaml` (the cron pitfall doc notes timezone is a top-level config key, not a per-job flag). All cron examples in §11 and §15 use this same form.
3. **Full rebuild, manual / on-recovery.** After an index loss, a Hindsight version bump, or a deployment-mode switch:

   ```bash
   PKM_VAULT=/opt/pkm-vault python3 ~/.hermes/hindsight/reindex_from_vault.py --full-rebuild
   ```

### Deployment modes (for reference / portability)

This spec uses the **pip `local_embedded`** path — the lightest fit for a single-user VPS. Because the index is rebuildable, switching modes later costs only a reindex run.

| Mode | How | When you'd switch |
|---|---|---|
| **pip / local_embedded** (this spec) | `hindsight-client` / `hindsight-server`, Postgres subprocess | Single-user VPS, vault canonical, Gemini free tier — **default** |
| **Docker** | API on **8888**, UI on **9999**, env `HINDSIGHT_API_LLM_API_KEY` + `HINDSIGHT_API_LLM_MODEL` | Index isolated in a container with its own Postgres |
| **Kubernetes / Helm** | Helm chart | Multi-user or HA — out of scope for a personal PKM |

Clients exist for Python, TypeScript/Node, Go, a CLI, and raw HTTP; the reindex job uses the Python/CLI client locally against bank `hermes`.

### Semantic recall vs. structural navigation — when to use which

Hindsight is **one of several** retrieval mechanisms over the vault, and deliberately not the first reached for. The vault's own structure is **primary**; Hindsight **complements** it.

| Mechanism | Strength | Used when |
|---|---|---|
| **`index.md`** (sectioned catalog) | Authoritative map of what exists | First stop / "what do I have on X?" — always read at session start |
| **`[[wikilinks]]` + graph** | Human-curated relationships | Navigating between known pages |
| **`tags`** | Precise faceting | Pulling all pages of a `type`/topic (also drives Bases) |
| **`search_files` (FTS/grep)** | Exact strings, acronyms, code | You know the term; large vaults where the index may miss content |
| **Hindsight `recall` (TEMPR)** | *Meaning* match | You don't know which page holds the answer, or phrasing differs from the page wording |
| **Hindsight `reflect`** | Cross-page synthesis | "What have I been learning?" — cron summaries, weekly digest |

### Cost (Gemini free-tier sizing)

Both extraction (`gemini-2.5-flash`) and embeddings (`text-embedding-004`, 3,072-dim) run on the **Gemini free tier** — ~60,000 `text-embedding-004` embeddings/month; the Anthropic key is never used here.

| Activity | Embeddings | Extraction calls | Approx. cost |
|---|---|---|---|
| Initial full reindex of 100 curated pages | ~100 | ~100 (small) | free-tier covered |
| Per-ingest incremental (1 source → 5–15 page touches) | 5–15 | 5–15 | negligible |
| Nightly catch-up (only *changed* pages) | <10 typical | same | negligible (unchanged skipped) |
| Daily search (10 recalls) | ~10 | 0 | ~$0.002 |
| Weekly `reflect` summary | ~50 | 1 large | ~$0.03 |

A personal vault of a few thousand pages with steady daily ingestion stays **comfortably inside the free tier**. A caveat on the "nightly runs cost almost nothing" claim: curated pages are the **churning layer** (the two-way edit model means the user edits pages directly in Obsidian, bumping `updated:` and the body), so the body-`sha256` skip only suppresses re-embed for pages *not* touched out-of-band. Nightly cost therefore scales with the number of pages edited in Obsidian since the last run — near-zero on quiet days, higher after a heavy editing session — **not** a flat ~zero. (`raw/` is immutable and carries the authoritative source `sha256`, but `raw/` is not indexed; the indexed curated layer is the one that churns.) To bound it if it ever bites: debounce re-embeds, or index a normalised fact-extract rather than the full marked-up body so cosmetic prose edits don't trigger re-embedding. The genuinely expensive event remains a **full rebuild** (one embedding per indexed page) — schedule those deliberately and rely on incremental runs day to day.

---

## 10. Life-Admin

Life-admin is a first-class part of the vault, not a bolt-on. TODOs/reminders, the media-to-consume backlog, and personal memories are ordinary wiki pages distinguished by their frontmatter `type` and surfaced as live Obsidian **Bases** dashboards on the Home page (§6). They are **not** a rival folder tree; the `type` field is the contract, the folder is an implementation detail that gives each page a real path (and thus a real `obsidian://` deep link).

### TODOs / reminders (`type: action`)

- Stored one file per task under `actions/`.
- Frontmatter: `status` (`todo|in_progress|done|completed|cancelled`), `priority` (`1 - Urgent` … `4 - Low`), `due_date` (`YYYY-MM-DD` or `…  HH:MM`), `recurrence` (`none|daily|weekly|monthly|yearly`), `project` (wikilink), `people` (list of `[[people/…]]`).
- Created from natural language ("remind me to…", "I need to…", "#project task"): the agent parses due date, recurrence, and priority and links the task to its project page.
- **Recurrence behaviour:** on completing a recurring action the agent re-opens it — sets `status: todo` and advances `due_date` by the `recurrence` interval — rather than spawning a new file, so history stays on one page. (This is agent behaviour, driven by the frontmatter contract.)
- Surfaced by `_bases/actions.base`: **Open Actions** (sorted by priority then due), **Due Soon** (next 7 days), **All Actions**. The daily briefing reads the same `status`/`due_date`/`priority` fields.

### Media backlog (`type: media`)

- One file per item under `media/`.
- Frontmatter: `media_type` (`book|film|tv|podcast|article|video|music`), `creator`, `url`, `status` (`to_consume|consuming|consumed`), `priority`.
- For a captured URL the agent extracts the real title (never "YouTube Video").
- Surfaced by `_bases/media.base`: **To Consume** and **In Progress or Done**. The daily summary nudges one or two backlog items; the weekly summary flags items going cold (`to_consume` > 180 days).

### Memories (`type: memory`)

- One file per memory/milestone under `memories/`.
- Frontmatter: `people` (list of `[[people/…]]`), `location`, `memory_date`.
- Personal/emotional captures (family, photos of people/places, holidays, milestones) route here, not to `concepts/`. A photo memory embeds its image inline (`![[asset.ext]]`) and describes it on the page.
- People mentioned link to `people/` pages — the entity sub-domain that ties memories and actions to the people involved.
- Surfaced by `_bases/memories.base`, sorted by `memory_date` descending.

---

## 11. Proactive Summaries

Two scheduled digests keep the vault *working for* the user. Both are **composed from the vault** (Actions/Media pages, recent `raw/` ingests, `log.md`/git history, and review-flagged frontmatter) — never from chat history or `state.db` — and delivered to the Slack DM. Both already exist as Hermes cron jobs and need only the content-source and model corrections below.

### Schedules (Hermes cron, Europe/London)

| Job | Cron expr | When | Model | Deliver |
|-----|-----------|------|-------|---------|
| `daily-summary` | `0 7 * * *` | every day 07:00 | `anthropic/claude-sonnet-4-6` | `slack:D0B61LKA3NV` |
| `weekly-summary` | `0 7 * * 1` | Monday 07:00 | `anthropic/claude-sonnet-4-6` | `slack:D0B61LKA3NV` |

> **Correction carried into the spec:** the live `daily-summary` job fails with `HTTP 404: model: claude-sonnet-4`, and `weekly-summary` is pinned to a non-resolving `gemini-pro`. Both must run on the valid pay-as-you-go Anthropic model id above so summaries actually fire — and must **never** use the Claude Max credential.

### Daily summary — content (sourced from the vault)

1. **Due / overdue Actions** — `type: action` pages whose `due_date` is today or past, ordered overdue-first (overdue flagged 🔴).
2. **Upcoming deadlines (next 3 days)** — `type: action` pages with `due_date` within the next 72h (Europe/London).
3. **Yesterday's ingests** — new/changed `raw/` sources and curated pages from the last day (read from `log.md` / git history), grouped by kind.
4. **Media-backlog nudge** — one or two `type: media` (to-consume) items worth picking up.
5. **Items flagged for review** — pages carrying a `review: true` / `status: review` frontmatter flag.

### Weekly summary — content (sourced from the vault)

1. **Week-in-review of ingests** — what entered `raw/` and which curated pages were created/expanded, with counts by kind.
2. **Emerging themes** — recurring tags/links/topics across the week's pages (may use Hindsight `reflect` over *curated* pages, never raw chatter).
3. **Actions status** — completion rate, what's still open, what's newly overdue.
4. **Upcoming week's deadlines** — all `type: action` `due_date`s in the next 7 days.
5. **Suggested review/archive** — stale or `review`-flagged pages, plus media-backlog items going cold.

### Example daily summary (delivered to Slack)

```
📋 Daily PKM Summary — Mon 2026-06-01 (Europe/London)

ACTIONS
  🔴 Overdue  — Reply to motor-control review     (due 2026-05-29)  [[actions/reply-motor-control]]
  🟡 Today    — Finish Q2 roadmap draft           (due today)       [[actions/q2-roadmap]]
  🟢 Next 3d  — Renew domain                       (due 2026-06-03)  [[actions/renew-domain]]

INGESTED YESTERDAY (4)
  • 2 articles  → distributed systems, event sourcing
  • 1 paper     → CAP theorem            [[cap-theorem]]
  • 1 image     → motor control diagram  [[program-motion-controller]]

MEDIA BACKLOG
  • Unread: "Designing Data-Intensive Applications" (added 12d ago)  [[media/ddia]]

FLAGGED FOR REVIEW
  • [[concepts/distributed-systems]] — 5 April cross-refs to re-check

Open any item in Obsidian via its wikilink, or ask me for a direct link.
```

Wikilinks in summaries double as retrieval handles: the user can ask "open the Q2 roadmap" and get the `obsidian://` link per §8.

---

## 12. Sync, Repos and Backup

### Repository topology

The system uses **two independent GitHub repositories** with different lifecycles, contents, and credentials. Today everything lives in the single `hermes-agent` config-backup repo; the migration (§16) splits the vault out.

| Repo | Visibility | Contains | Written by | Sync cadence | Durable backup? |
|------|-----------|----------|------------|--------------|-----------------|
| `pkm-vault` (new, dedicated) | Private | The Obsidian vault only: `raw/`, `entities/`, `concepts/`, `comparisons/`, `queries/`, life-admin pages, `SCHEMA.md`, `index.md`, `log.md`, `.obsidian/` | VPS agent (after ingests) **and** workstation Obsidian Git plugin | Continuous two-way git | **Yes** — git history is the point-in-time snapshot store |
| `hermes-agent` (existing) | Private | Agent config (`config.yaml`, `SOUL.md`, `skills/`, `memories/`, `cron/jobs.json`, `hindsight/config.json`), periodic `db-snapshots/*.db` | VPS only (config-backup cron) | Push every 6h | Yes — for *config + transient DB*, not knowledge |

Rationale for the split: the vault is edited from two writers and needs **two-way** sync; agent config is single-writer (VPS) and only needs push-only backup. The vault must stay small and text-clean for fast Obsidian Git operation; mixing 14 MB `state.db` snapshots into it would bloat history and slow every pull/push on the phone. Different blast radius: a fine-grained PAT scoped to `pkm-vault` cannot touch agent secrets/config, and vice-versa.

```
VPS  (/opt/hermes-agent + /opt/pkm-vault)     Workstation (Obsidian)
┌───────────────────────────────┐             ┌──────────────────────────┐
│ pkm-vault/  (git clone)        │  push/pull  │ pkm-vault/  (git clone)  │
│   raw/ entities/ concepts/ ... │ ←─────────→ │   + Obsidian Git plugin  │
│   agent: pull-before-write,    │   origin    │   auto pull/commit/push  │
│   commit+push after ingest     │ github.com  │                          │
├───────────────────────────────┤  :pkm-vault └──────────────────────────┘
│ hermes-agent/ (git clone)      │
│   config + db-snapshots/       │  push only ──→ github.com:hermes-agent
└───────────────────────────────┘
```

### Two-way git sync design

The vault is a normal git working tree on both ends — no rsync, no bespoke daemon, no cloud-drive mount, just git (free conflict detection + complete history).

#### Vault `.gitignore`

```gitignore
# pkm-vault/.gitignore — keep the vault clean and conflict-free
.obsidian/workspace.json        # per-device pane layout; churns constantly
.obsidian/workspace-mobile.json
.obsidian/cache
.trash/
.DS_Store
*.tmp
.obsidian/plugins/obsidian-git/.gitignore
```

Track the rest of `.obsidian/` (notably `plugins/obsidian-git/data.json` and `core-plugins.json`) so a freshly cloned device inherits the same plugin configuration. Exclude only the per-device `workspace*.json` and caches — the files that otherwise produce noise commits and spurious conflicts.

#### Workstation side — Obsidian Git community plugin

Install **Obsidian Git** (`denolehov/obsidian-git`) from Community Plugins. It performs scheduled `pull → stage-all → commit → push` against `origin`. Recommended settings:

| Setting | Value | Why |
|---|---|---|
| Vault backup interval (minutes) | `10` | Commit+push every 10 min when dirty |
| Auto pull interval (minutes) | `10` | Pull agent ingests roughly as often as you push |
| Pull updates on startup | `on` | First action on open is to fetch overnight writes |
| Push on backup | `on` | Backups are useless if they never leave the device |
| Commit message | `vault backup {{date}} ({{hostname}})` | Distinguishes workstation vs phone vs agent commits |
| Sync method | `merge` *(default)* | rebase is acceptable too |
| Pull before push | `on` | Avoids the trivial "fetch first" rejection |
| List changed files in commit body | `on` | Human-auditable history |
| Notifications | keep on | You *want* to see when a sync conflicts |

In **Settings → Files & Links**, enable **"Detect all file changes"** — mandatory. By default Obsidian only watches files it thinks are notes; without this, externally written files (the agent's commits under `raw/assets/`, new `.md` pages) are not detected until a manual reload, and Obsidian Git can miss them when staging. On mobile, set the backup interval to `15` and rely primarily on app-foreground pull.

#### VPS agent side — pull-before-write, commit-and-push-after-ingest

Two helper scripts wrap every vault mutation. They use `gh`'s credential helper for auth (never SSH, never a PAT-in-URL) and `GIT_CONFIG_GLOBAL=/dev/null` so any global `insteadOf` ssh-rewrite cannot hijack the HTTPS remote.

`/opt/hermes-agent/bin/vault-pull` (run before any write):

```bash
#!/usr/bin/env bash
set -euo pipefail
VAULT="${PKM_VAULT:-/opt/pkm-vault}"
GIT_CONFIG_GLOBAL=/dev/null git -C "$VAULT" \
    -c credential.helper='!gh auth git-credential' \
    pull --rebase --autostash origin main
```

`/opt/hermes-agent/bin/vault-commit` (run after an ingest batch completes):

```bash
#!/usr/bin/env bash
set -euo pipefail
VAULT="${PKM_VAULT:-/opt/pkm-vault}"
MSG="${1:-ingest $(date -u +%Y-%m-%dT%H:%MZ)}"
cd "$VAULT"
git add -A
git diff --cached --quiet && { echo "nothing to commit"; exit 0; }
# Commit FIRST so the working tree is clean, THEN rebase onto any Obsidian pushes that
# landed during the batch. Committing before the rebase means no autostash is needed, so a
# conflicting Obsidian push surfaces as a clean rebase conflict (fail loudly) instead of a
# half-applied mid-stash-pop state. The agent never --force pushes.
git commit -m "agent: $MSG"
if ! GIT_CONFIG_GLOBAL=/dev/null git -c credential.helper='!gh auth git-credential' \
        pull --rebase origin main; then
    echo "VAULT CONFLICT during rebase — resolve in Obsidian; not pushing." >&2
    GIT_CONFIG_GLOBAL=/dev/null git rebase --abort || true
    exit 1   # wrapper surfaces the conflicting path over Slack; never clobber
fi
GIT_CONFIG_GLOBAL=/dev/null git -c credential.helper='!gh auth git-credential' \
    push https://github.com/<owner>/pkm-vault.git main
```

The PKM skills call `vault-pull` at the start of any ingest/curation turn and `vault-commit "<topic>"` once the page(s) and any `raw/assets/` binaries are written. Because the agent is single-process and serialises its turns, it never races itself; the only other writer is Obsidian on the workstation.

#### Conflict strategy

Collisions are structurally rare and the design leans into that:

- **`raw/` is immutable.** Uploaded sources are written once and never rewritten, so two writers can never edit the same raw file. New uploads land under unique, content-addressed names.
- **One file per topic.** The agent edits the page for the thing it just ingested while the human is typically editing a *different* page.
- **Rebase/merge, autostash.** Both ends pull with `--rebase --autostash` (agent) or `merge` (Obsidian). For disjoint files git fast-forwards or auto-merges with zero intervention.
- **On the occasional real conflict:** git leaves standard `<<<<<<<`/`=======`/`>>>>>>>` markers in the one affected `.md`. `vault-pull` fails loudly (non-zero exit) rather than clobbering; the wrapper surfaces the path over Slack ("merge conflict in `concepts/raft.md`, resolve in Obsidian"). The human resolves in Obsidian (markdown is human-mergeable); the next auto-commit clears it. The agent must **never** `--force` push.
- **Attachments / binaries.** Images and PDFs under `raw/assets/` are immutable and uniquely named, so they only ever *add*, never conflict.

#### GitHub auth (both repos)

HTTPS + a **fine-grained PAT** via `gh`'s credential helper, per the user's global rule. Never SSH, never embed the PAT in a remote URL.

```bash
echo "$PAT" | gh auth login --with-token   # fine-grained PAT
gh auth setup-git                            # install gh as git credential helper for github.com
```

The vault PAT is scoped to **only** `pkm-vault` with **Contents: Read & write** + **Metadata: Read-only**. The config-backup PAT is a separate token scoped to **only** `hermes-agent`. Simplest correct posture: one fine-grained PAT with access to exactly these two repositories and nothing else.

**Repo-creation caveat.** A fine-grained PAT scoped to two *pre-existing* repos with Contents/Metadata **cannot create new repos** — so `gh repo create pkm-vault` (Phase C / §16 step 2) will fail with this steady-state token. Either create `pkm-vault` once in the GitHub web UI, or use a broader token (Administration/repo-creation permission) for that one-time create and then switch to the narrow two-repo PAT for steady-state push/pull. Also verify `gh auth login --with-token` accepts the fine-grained PAT on the installed `gh` version.

**Expiry and rotation.** Fine-grained PATs expire (set a **90-day expiry**). Because the agent's vault push and config backup both depend on a non-expired token, schedule a **rotation reminder before expiry**: a recurring Hermes cron job (or an Action page under `actions/` with `recurrence: monthly`) that pings Slack ~1 week before the 90-day mark to re-issue both PATs in the GitHub UI and refresh them via `gh auth login --with-token`. An expired PAT silently breaks unattended pushes, so treat rotation as an operational task, not an afterthought. (Mirrors the absorbed runbook's rotation reminder.)

### Backup and recovery

The backup model follows from the source-of-truth decision: **the vault repo *is* the durable backup**, Hindsight is disposable, and agent config has its own separate push-only backup.

| Asset | Backup mechanism | Recoverable from | RPO |
|---|---|---|---|
| Knowledge (vault markdown + `raw/assets/` binaries) | `pkm-vault` git history on GitHub + every cloned device | `git clone`; any commit = a point-in-time snapshot | ≤ minutes |
| Hindsight semantic index (local Postgres, `bank_id=hermes`) | **Not backed up — rebuilt** | the reindex job | n/a (regenerated) |
| Agent config (`config.yaml`, `SOUL.md`, `skills/`, `memories/`, `cron/jobs.json`, `hindsight/config.json`) | `hermes-agent` repo, config-backup cron | `git clone` of `hermes-agent` | 6h |
| Transient working memory (`state.db`, `kanban.db`) | `db-snapshots/*.db` in `hermes-agent` (online `.backup`) | restore snapshot — **only** in-flight sessions/kanban, *not* knowledge | 6h |
| Secrets (`~/.hermes/.env`) | **Never** in any repo — password manager only | manual re-entry | n/a |

**Key consequence:** losing `state.db` loses nothing canonical. Its snapshot exists only to avoid re-opening in-flight chats and to preserve the Kanban board — not to preserve knowledge. Knowledge is safe in the vault repo.

**Full recovery from a lost VPS:**

1. Provision a new VPS (Ubuntu 24.04+, 2 cores, 8 GB RAM, 50 GB+ disk).
2. Run Setup Runbook Phases A–B (prereqs + Hermes install, §15).
3. Restore agent config (clone with the inline `gh` helper + nulled global config, so the user's `insteadOf` ssh-rewrite cannot hijack the HTTPS URL):
   ```bash
   echo "$PAT" | gh auth login --with-token
   GIT_CONFIG_GLOBAL=/dev/null git -c credential.helper='!gh auth git-credential' \
     clone https://github.com/<owner>/hermes-agent.git /opt/hermes-agent-config
   mkdir -p ~/.hermes && cp -r /opt/hermes-agent-config/{config.yaml,SOUL.md,skills,memories,cron,hindsight} ~/.hermes/
   ```
4. Clone the canonical vault (the knowledge restore — nothing else is needed), same inline-helper pattern:
   ```bash
   GIT_CONFIG_GLOBAL=/dev/null git -c credential.helper='!gh auth git-credential' \
     clone https://github.com/<owner>/pkm-vault.git /opt/pkm-vault
   ```
5. Re-add secrets manually from the password manager (`~/.hermes/.env`, `chmod 600`).
6. **Rebuild the index** from the vault: `PKM_VAULT=/opt/pkm-vault python3 ~/.hermes/hindsight/reindex_from_vault.py --full-rebuild`. (Do **not** restore `state.db` as the knowledge base; an optional snapshot restore only recovers in-flight sessions/Kanban.)
7. Re-enable systemd + cron (§15 Phases F, G), then run the verification checklist.

**Estimated recovery time:** ~1–2 h, dominated by package installs and the reindex pass; the knowledge itself is restored the instant the vault clone completes.

**Scaling note.** Plain git is good to ~1 GB. While the vault stays text-heavy with a modest number of images, commit binaries directly. When `raw/assets/` growth pushes the repo toward ~1 GB, migrate binaries to **Git LFS** (10 GB free) or move the asset tree to **restic → Backblaze B2** (encrypted, deduplicated) while keeping the markdown in plain git. Obsidian Git on mobile has known reliability limits with large binary pulls, so the mobile path may force this graduation earlier than the desktop. This is a later optimisation, not an upfront requirement.

---

## 13. Maintenance

A lint operation runs on demand and as a scheduled job (its own cron job — not the `auxiliary.curator` helper, not the top-level `curator` skills block, both of which are unrelated to the vault). It is a programmatic scan across all `.md` files, reporting grouped by severity and appending one `log.md` entry.

| # | Check | Action |
|---|---|---|
| 1 | **Orphan pages** | Knowledge pages with zero inbound `[[wikilinks]]` (life-admin pages exempt — Bases surface them). Suggest a link or archive. |
| 2 | **Broken wikilinks** | `[[link]]` targets that resolve to no file (respect `aliases`). Highest severity. |
| 3 | **Index completeness** | Every knowledge page appears in `index.md`; flag missing and stale "Total pages". |
| 4 | **Frontmatter validation** | Required common fields present; `type` valid; type-specific required fields present (e.g. `action` has `status`); values match Metadata Menu vocab. |
| 5 | **Stale content** | Knowledge `updated` > 90 days older than the newest source touching the same entities; **life-admin**: `action` past `due_date` and not done/cancelled; `media` `to_consume` > 180 days. |
| 6 | **Contradictions** | Surface every page with `contested: true` or non-empty `contradictions:`; flag same-topic pages stating different facts. |
| 7 | **Source drift** | For each `raw/` file with `sha256:`, recompute over body; mismatch ⇒ raw edited (shouldn't happen) or source URL changed. Report, don't hard-fail. |
| 8 | **Quality signals** | List `confidence: low` and single-source pages with no `confidence` — corroborate or demote. |
| 9 | **Page size** | Knowledge pages > 200 lines ⇒ split candidates. |
| 10 | **Tag audit** | Every tag in use must exist in SCHEMA.md taxonomy; flag strays. |
| 11 | **Image hygiene** | Assets in `raw/assets/` with no `![[…]]` embed anywhere = orphan binaries; pages embedding a missing asset = broken embed; any surviving per-image sidecar `.md` (legacy pattern) flagged for merge-into-owner-page. |
| 12 | **Log rotation** | If `log.md` > 500 entries, rotate to `log-YYYY.md`. |
| 13 | **Report + log** | Group by severity (broken links/embeds > orphans > source drift > contested > stale/overdue > style). Append `## [YYYY-MM-DD] lint | N issues found`. |

The lint job runs weekly via Hermes cron (§15 Phase G), and on demand after a bulk migration or restore. The vault `reindex` (§9) is a separate weekly job; the two are independent.

---

## 14. Security and Privacy

The VPS is isolated and runs at high autonomy (`approvals.mode: auto`), so the guard-rails below are what make that posture safe.

### LLM credential handling — the Anthropic OAuth ban risk

- The agent authenticates to Anthropic with a **pay-as-you-go API key** (`ANTHROPIC_API_KEY`) from `console.anthropic.com`, billed independently of the Claude Max subscription.
- **Never** drive the agent with the Claude Max/Pro **OAuth token.** Programmatic/agentic use of subscription OAuth credentials violates Anthropic's terms and carries an **account-ban risk**. The API key and the subscription must stay strictly separated; the VPS only ever sees the API key. (This also keeps the runaway-cost blast radius on a metered key you can cap.)
- Gemini (embeddings/extraction) uses a free-tier key with no billing; Exa and Firecrawl use their own keys. All keys live only in `~/.hermes/.env`, `chmod 600`, **never** committed — `.env` is gitignored and excluded from every backup path.

### Secrets at rest and in transit

- `~/.hermes/.env` is the single secrets file; the password manager is the only off-box copy.
- `security.redact_secrets: true` scrubs secret-looking strings from model I/O and logs, so a pasted token isn't echoed back or persisted into a vault page.
- The vault and its history must stay secret-free: the agent writes *curated knowledge*, not raw chat transcripts, and redaction runs before anything is filed. Treat the private `pkm-vault` repo as confidential (redaction is the real control).

### MCP remote-exposure auth

MCP is the secondary interface. It must **not** be exposed as bare HTTP on a public port:

- **Local / Claude Code:** stdio MCP — no network surface at all.
- **Remote (claude.ai):** front the MCP HTTP/SSE endpoint with a **Cloudflare Tunnel** (no public IP, no inbound firewall hole, TLS terminated by Cloudflare, optional Cloudflare Access identity gating). An SSH `-L` tunnel is an acceptable manual per-session alternative. A bare reverse-proxy-on-public-IP is **not** acceptable unless it enforces a bearer token *and* TLS — Cloudflare Tunnel is strictly preferred because it removes the public attack surface entirely.

### Command/write guard-rails and scanning

- **Write deny list / autonomy bounds.** Even at `approvals.mode: auto`, keep `command_allowlist` minimal (currently just stop/restart system service) so destructive host actions still gate. The agent's vault writes are confined to the `pkm-vault` working tree; it has no business writing outside `~/.hermes` and the vault clone. The force-push-never policy is part of this: the agent may add/commit/push but is forbidden from history-rewriting operations (`push --force`, `reset --hard` against `origin`, `filter-branch`).
- **Tirith static scanning.** `security.tirith_enabled: true` scans agent-generated/edited code and commands. The current config sets `tirith_fail_open: true` — on an isolated single-user box this is a deliberate availability trade-off, but for a hardened posture consider `tirith_fail_open: false` so a scanner outage blocks rather than waves through risky actions.
- **Network egress.** `security.allow_private_urls: false` blocks SSRF-style fetches to private/loopback ranges via the web tools (Exa/Firecrawl). Keep it false.
- **Isolated-VPS posture.** The box runs only this agent; high autonomy is acceptable *because* the blast radius is one dedicated host with no lateral access, metered API keys, and a fully rebuildable index + git-versioned vault. The systemd unit should ideally run as a dedicated unprivileged `hermes` user (the Hermes container uses UID 10000) rather than `root`; see §15.

---

## 15. Setup and Operations Runbook

This consolidates and corrects the prior `SETUP-RUNBOOK.md` for the new architecture (separate vault repo, Obsidian Git, no DB-as-truth). Phases A–B (system packages, `uv`, Node, Hermes install/venv, PATH) are unchanged from that document and referenced rather than repeated.

**Phase A — Prereqs & Hermes.** Install `sqlite3 git jq build-essential libffi-dev libssl-dev`, `uv`, Node 20+, clone `hermes-agent`, create venv (`uv venv --python 3.14`, fallback 3.12/3.11), `uv pip install -e ".[all,dev]"`, add venv to PATH. Verify `hermes --version` and `hermes doctor`.

**Phase B — Secrets & tokens.** Create `~/.hermes/.env` (`chmod 600`) and populate the credential families:

| Token | Where | `.env` var | Scope/notes |
|---|---|---|---|
| Anthropic LLM | console.anthropic.com (pay-as-you-go) | `ANTHROPIC_API_KEY` | **API key only — never the Max OAuth token** |
| Gemini embeddings/extraction | aistudio.google.com | `GOOGLE_GEMINI_API_KEY` (+ `HINDSIGHT_LLM_API_KEY`) | Free tier; drives Hindsight `llm_provider: gemini` |
| Slack bot + app | api.slack.com/apps | `SLACK_BOT_TOKEN` (`xoxb-`), `SLACK_APP_TOKEN` (`xapp-`) | Socket Mode; scopes `chat:write files:read files:write im:history im:read im:write`; subscribe `message.im`, `file_shared` |
| Exa / Firecrawl | dashboard.exa.ai / firecrawl.dev | `EXA_API_KEY`, `FIRECRAWL_API_KEY` | Web search + extract |
| GitHub PAT (vault) | github.com fine-grained | via `gh auth login --with-token` | **Only `pkm-vault`**, Contents R/W + Metadata RO |

```bash
echo "$PAT" | gh auth login --with-token && gh auth setup-git
gh auth status
```

> **PAT expiry/rotation (§12).** Issue the fine-grained PAT(s) with a **90-day expiry** and set a rotation reminder (recurring cron or an `actions/` page with `recurrence: monthly`) to re-issue + re-`gh auth login --with-token` ~1 week before expiry — an expired token silently breaks the unattended vault push and config backup.
>
> **`gh auth setup-git` scope.** This installs `gh` as the credential helper in the **global** git config (`~/.gitconfig`). It helps `gh` API calls and any plain `git` that reads global config — but the vault wrapper scripts run with `GIT_CONFIG_GLOBAL=/dev/null` (to defeat the user's `insteadOf` ssh-rewrite), so they do **not** see this helper and instead pass `-c credential.helper='!gh auth git-credential'` inline. Both are needed: `setup-git` for `gh repo create` etc., the inline helper for the wrappers.
>
> **Reuse the existing Slack app.** The deployment already has an installed Slack app (committed `slack-manifest.json`; App Home is the primary surface). **Reuse it and its existing `xoxb-`/`xapp-` tokens** — do not recreate. Confirm **App Home is enabled** (so the DM is the primary surface) and treat the scope list above as a *verification checklist* against `slack-manifest.json`, not a from-scratch app creation.

**Phase C — Create and initialise the vault repo.**

```bash
# NOTE: `gh repo create` needs a repo-creation-capable token; the narrow two-repo PAT cannot
# create repos (§12). Create pkm-vault in the GitHub UI, or use a broader token for THIS step only.
gh repo create pkm-vault --private --description "Hermes PKM Obsidian vault (canonical)"
# Clone with the inline gh helper + nulled global config (defeats the user's insteadOf ssh-rewrite):
GIT_CONFIG_GLOBAL=/dev/null git -c credential.helper='!gh auth git-credential' \
  clone https://github.com/<owner>/pkm-vault.git /opt/pkm-vault
cd /opt/pkm-vault
mkdir -p raw/{articles,papers,transcripts,assets} \
         entities concepts comparisons queries \
         actions media memories people \
         _bases _meta _archive inbox .obsidian
printf '%s\n' '.obsidian/workspace.json' '.obsidian/workspace-mobile.json' \
  '.obsidian/cache' '.trash/' '.DS_Store' '*.tmp' > .gitignore
# Seed the three llm-wiki spine files (SCHEMA.md, index.md, log.md) per §6, and the _bases/*.base files per §6
touch SCHEMA.md index.md log.md
git add -A && git commit -m "init: vault skeleton (raw/ + curated layers + life-admin + spine)"
GIT_CONFIG_GLOBAL=/dev/null git -c credential.helper='!gh auth git-credential' \
  push -u https://github.com/<owner>/pkm-vault.git main
export PKM_VAULT=/opt/pkm-vault   # persist in the systemd unit's Environment=
```

**Phase D — Configure Obsidian Git on the workstation.**

1. Clone `pkm-vault` locally; `File → Open vault → Open folder as vault`. Confirm the vault registers as **`pkm-vault`** (must match `OBSIDIAN_VAULT_NAME` exactly for deep links).
2. **Settings → Files & Links → Detect all file changes: ON** (mandatory).
3. **Settings → Community plugins → Browse → Obsidian Git → Install → Enable.**
4. Apply the §12 settings (10-min backup, 10-min auto-pull, pull-on-startup ON, push-on-backup ON, pull-before-push ON).
5. Commit `.obsidian/plugins/obsidian-git/` so other devices inherit them.
6. Confirm round-trip: have the agent file a test page, wait one pull interval, see it in Obsidian; edit it in Obsidian, confirm the agent sees the change after `vault-pull`.

**Phase E — Hermes memory/Hindsight config.** Confirm `config.yaml` has `memory.provider: hindsight` and `hindsight/config.json` has `mode: local_embedded`, `bank_id: hermes`, `llm_provider: gemini`, **`auto_retain: false`**, and `retain_context: "facts already filed into the Obsidian vault by Hermes"` (§9). Seed the index: `PKM_VAULT=/opt/pkm-vault python3 ~/.hermes/hindsight/reindex_from_vault.py --full-rebuild`.

**Phase F — gateway service (use the Hermes-managed unit; do not hand-roll ExecStart).** The as-built deployment registers the gateway with `hermes gateway install`, which generates and enables a systemd unit named **`hermes-gateway.service`** and is controlled via `hermes gateway start|stop|restart|status`. **Keep this tool-generated name** — there is no separate `hermes-pkm.service`; the spec standardises on the as-built `hermes-gateway.service` everywhere to avoid two competing units and a fabricated `ExecStart`/`--gateway slack` form that the installed CLI does not use.

First make the vault env vars visible to the gateway. The skills do **not** auto-alias `PKM_VAULT`: the llm-wiki skill reads `WIKI_PATH` (default `~/wiki`) and the obsidian skill reads `OBSIDIAN_VAULT_PATH` (default `~/Documents/Obsidian Vault`). **Each must be set explicitly to `/opt/pkm-vault`** or those skills write to the wrong place. Put them in `~/.hermes/.env` (loaded by the gateway):

```bash
# ~/.hermes/.env — vault path must be set for EVERY skill that reads it (no auto-alias)
PKM_VAULT=/opt/pkm-vault
WIKI_PATH=/opt/pkm-vault
OBSIDIAN_VAULT_PATH=/opt/pkm-vault
OBSIDIAN_VAULT_NAME=pkm-vault
```

Then install and start the gateway via the Hermes CLI (which writes the unit; do not author one by hand):

```bash
hermes gateway install      # generates + registers hermes-gateway.service (boot-enabled)
hermes gateway start
hermes gateway status        # or: systemctl status hermes-gateway.service
journalctl -u hermes-gateway.service -n 50 --no-pager
```

> **User context.** `hermes gateway install` registers the unit for the invoking user. As-built this is `root` (`HERMES_HOME=/root/.hermes`). The hardened target is a dedicated unprivileged `hermes` user (`HERMES_HOME=/home/hermes/.hermes`); if you defer creating it (Open Question #9), the root paths apply. Either way the unit name stays `hermes-gateway.service`. Verify the exact env-injection mechanism (`hermes gateway install` may read `~/.hermes/.env` directly, or you may add an `EnvironmentFile=` drop-in at `/etc/systemd/system/hermes-gateway.service.d/override.conf`).

**Phase F.1 — MCP server (secondary interface, alongside the gateway).** Hermes exposes a native MCP server over **stdio** (`hermes mcp serve`) — no wrapper, no network surface. This satisfies delta item #7 (MCP stood up, not deferred). It runs in addition to (not instead of) the Slack gateway: the gateway is the always-on `hermes-gateway.service`; the stdio MCP is launched per-session by the MCP client (below), so the two coexist without a second daemon.

For **Claude Code** (local stdio — the v1 target), add to `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "hermes-pkm": {
      "command": "/opt/hermes-agent/venv/bin/hermes",
      "args": ["mcp", "serve"],
      "env": { "HERMES_HOME": "/root/.hermes", "PKM_VAULT": "/opt/pkm-vault",
               "WIKI_PATH": "/opt/pkm-vault", "OBSIDIAN_VAULT_PATH": "/opt/pkm-vault",
               "OBSIDIAN_VAULT_NAME": "pkm-vault" }
    }
  }
}
```

(`command` is the `hermes` binary path — `which hermes`; on a one-liner install it is `/usr/local/bin/hermes`, on this venv install `/opt/hermes-agent/venv/bin/hermes`. Set `HERMES_HOME` to match the gateway user. Restart Claude Code to load the server.) Claude Code launches the stdio server **on demand per session**, so it needs no separate systemd unit; there is no always-on MCP process to supervise for the local case.

**Tools exposed over MCP** (the PKM capability surface — same operations as Slack, text-in/text-out): `pkm_ingest` (capture + file a curated page), `pkm_search` (retrieve → vault pages + `obsidian://` links/paths/wikilinks), `pkm_todos` (open/due Actions from `actions/`), `pkm_recent` (recent captures/ingests). MCP retrieval returns paths + wikilinks (no Slack-style upload).

Only the **remote** exposure for claude.ai (HTTP/SSE behind a Cloudflare Tunnel, §14) stays an open item (Open Question #10) — the local stdio MCP above is fully specified here.

**Phase G — Cron jobs (verified `cron create` form).** Use the CLI form the as-built `hermes-pkm-ops` skill documents: **`hermes cron create "<schedule>" "<prompt>" --name … --model … --deliver …`** — schedule and prompt are **positional** (schedule first, prompt second), flags after; `--model` takes a **JSON object**, not a slash-string. Timezone comes from the global `timezone: Europe/London` in `config.yaml` (a top-level key, not a per-job flag). Set **valid model IDs** — the as-built jobs fail with `HTTP 404: model: claude-sonnet-4` (daily) and an unresolved `gemini-pro` (weekly). On the model id, see the verification note below the block.

```bash
hermes cron create "0 7 * * *" \
  "Daily PKM summary FROM THE VAULT: Actions due today + overdue, deadlines next 3 days, recap of yesterday's ingests (log.md/git), one media-backlog nudge, review-flagged pages. Link items via [[wikilinks]] / obsidian:// deep links. Be concise." \
  --name "daily-summary" \
  --model '{"model":"claude-sonnet-4-6","provider":"anthropic"}' \
  --deliver "slack:D0B61LKA3NV"

hermes cron create "0 7 * * 1" \
  "Weekly PKM summary FROM THE VAULT: week-in-review of ingests, emerging themes (hindsight reflect over curated pages), Actions completion + outstanding, next-week deadlines, items to review/archive. Be concise." \
  --name "weekly-summary" \
  --model '{"model":"claude-sonnet-4-6","provider":"anthropic"}' \
  --deliver "slack:D0B61LKA3NV"

# Nightly Hindsight reindex (catch out-of-band Obsidian edits), 06:30 before the 07:00 summary.
# MUST pin an explicit Gemini model (else inherits a Claude name on the Gemini provider -> HTTP 404):
hermes cron create "30 6 * * *" \
  "vault-pull, then run reindex_from_vault.py (incremental) to sync Hindsight with pages edited directly in Obsidian overnight." \
  --name "hindsight-reindex" \
  --model '{"model":"gemini-2.5-flash","provider":"gemini"}' \
  --deliver "slack:D0B61LKA3NV"

# Weekly vault lint (maintenance):
hermes cron create "0 8 * * 1" \
  "Run the vault lint (§13): report broken links/embeds, orphans, source drift, contested, stale/overdue, tag strays. Append a log.md entry." \
  --name "vault-lint" \
  --model '{"model":"claude-sonnet-4-6","provider":"anthropic"}' \
  --deliver "slack:D0B61LKA3NV"
```

> **Model-id verification (do this before standardising).** `claude-sonnet-4-6` is **attested running in this deployment** (it appears as `model=claude-sonnet-4-6` in the as-built gateway logs), which is why it is the target. But the only *proven-failing* mode is an unresolved alias (`claude-sonnet-4` → 404), so confirm `claude-sonnet-4-6` resolves with `hermes model` / the model catalog before relying on it. **Safe fallback: the dated pin `claude-sonnet-4-20250514`** (the current as-built working value) — substitute it in every job and in `config.yaml` if `-4-6` does not resolve on your build. Do not assume bare aliases resolve; Anthropic generally requires dated ids.

```cron
# system crontab — config-backup repo every 6h (push-only; quiesces the transient DB)
0 */6 * * * /opt/hermes-agent/bin/config-backup.sh >> /var/log/hermes-config-backup.log 2>&1
```

`config-backup.sh` is the as-built `backup.sh` (under the `hermes-pkm-ops` skill) promoted to a delivered artefact and renamed; its full body (modelled on the skill's `references/backup.sh`, hardened with the `gh` credential helper + `GIT_CONFIG_GLOBAL=/dev/null` per the user's global git rule):

```bash
#!/usr/bin/env bash
# /opt/hermes-agent/bin/config-backup.sh
# Push-only backup of agent config + transient-DB snapshots to the hermes-agent repo.
# Secrets (.env, auth.json) are NEVER committed — excluded via ~/.hermes/.gitignore.
set -euo pipefail

HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"
TS="$(date -u +%Y-%m-%dT%H:%MZ)"

# 1. Online, consistent SQLite snapshots of the TRANSIENT working-memory DBs.
mkdir -p "$HERMES_HOME/db-snapshots"
for DB in state.db kanban.db; do
  if [ -f "$HERMES_HOME/$DB" ]; then
    sqlite3 "$HERMES_HOME/$DB" ".backup '$HERMES_HOME/db-snapshots/$DB'"
    echo "[$TS] snapshotted $DB"
  fi
done

# 2. Commit + push to the hermes-agent config-backup repo (NOT the vault repo).
#    .env / secrets stay gitignored; the vault is untouched (it has its own per-ingest push).
cd "$HERMES_HOME"
git add -A
if git diff --cached --quiet; then
  echo "[$TS] no config/DB changes to back up"
else
  git commit -m "backup $TS"
  GIT_CONFIG_GLOBAL=/dev/null git -c credential.helper='!gh auth git-credential' \
    push https://github.com/<owner>/hermes-agent.git main
  echo "[$TS] config backup pushed"
fi
```

The vault is *not* touched here — the vault has its own per-ingest push (§12). This script only ever writes to the `hermes-agent` repo.

**Phase H — Verify.** Ingest a doc → confirm a curated vault page appears in `pkm-vault` (git log) *and* in Obsidian after one pull; retrieve a doc → confirm Slack returns a clickable `obsidian://open?vault=pkm-vault&file=<url-encoded-path>` link; kill and reboot the VPS → confirm full recovery via the §12 steps.

---

## 16. Migration Plan

Today's state (verified in-repo): a `documents/` proto-vault with `images/` holding three binaries and **separate sidecar `.md` descriptions** (`img_e4a408e064c4.png` + `img_e4a408e064c4.md`; `img_ed814b035583.png` + `img_ed814b035583.md`), all committed inside the single `hermes-agent` config-backup repo; `state.db` (≈14.5 MB) + `kanban.db` snapshots in `db-snapshots/`; and `SOUL.md` asserting "**the DB is the truth**". The third image `img_85e18fa08300.jpeg` has **no sidecar** (orphan to fix). Migration converts each sidecar into a curated page that *embeds* its image, relocates binaries into `raw/assets/`, generates the wiki spine, reindexes, splits the vault into its own repo, and retires the DB-as-truth model.

Execute these numbered, individually verifiable steps:

**1. Snapshot the current state (safety net).**
```bash
git -C /workspaces/hermes-agent tag pre-migration && \
git -C /workspaces/hermes-agent commit -am "checkpoint: pre-vault-migration" || true
```
*Verify:* `git tag` lists `pre-migration`.

**2. Create and clone the empty private vault repo** (Phase C).
*Verify:* `pkm-vault` exists on GitHub and clones clean.

**3. Lay down the directory skeleton + `.gitignore`** in the clone (Phase C trees).
*Verify:* `raw/{articles,papers,transcripts,assets} entities concepts comparisons queries actions media memories people _bases _meta _archive inbox` all present.

**4. Move binaries into `raw/assets/` (immutable layer)**, renaming to descriptive slugs.
```bash
cp /workspaces/hermes-agent/documents/images/img_e4a408e064c4.png \
   /opt/pkm-vault/raw/assets/motor-control-diagram-e4a408.png
cp /workspaces/hermes-agent/documents/images/img_ed814b035583.png \
   /opt/pkm-vault/raw/assets/ioc-network-diagram-ed814b.png
cp /workspaces/hermes-agent/documents/images/img_85e18fa08300.jpeg \
   /opt/pkm-vault/raw/assets/uncategorized-image-85e18f.jpeg   # orphan: no sidecar; neutral slug until the agent describes it
```
*Verify:* all three images exist under `raw/assets/` with descriptive names. The two diagrams (`e4a408`, `ed814b`) carry topic slugs; the orphan jpeg (`85e18f`) gets a neutral `uncategorized-image-` slug because it is **not** a motor-control diagram and its subject is unknown until step 5 generates a description.

**5. Convert each sidecar `.md` into a curated page that embeds its image.** The curated page both *describes* and *embeds* the image via an Obsidian wiki-embed; no base64; no surviving sidecar. Place pages by subject — these diagrams are `entities/` (named systems). Worked pages are in §18 (Appendix). Map:

| Old sidecar | New curated page | Transform |
|---|---|---|
| `img_e4a408e064c4.md` | `entities/program-motion-controller.md` | fold description into an entity page that **embeds** `motor-control-diagram-e4a408.png`; delete sidecar |
| `img_ed814b035583.md` | `concepts/ioc-network-architecture.md` | embed `ioc-network-diagram-ed814b.png`; preserve its "Questions Noted" as an **Open questions** section; delete sidecar |
| `img_85e18fa08300.jpeg` (orphan, README reads "[Needs description]") → asset `raw/assets/uncategorized-image-85e18f.jpeg` | `inbox/uncategorized-image-85e18f.md` (a `type: inbox` stub) | embed `![[uncategorized-image-85e18f.jpeg]]`, have the agent vision-describe it, then refile to the right `entities/`/`concepts/`/`memories/` page once the subject is known — so no asset is left undocumented. This is **not** PMC's diagram (that is `e4a408`). |

*Verify:* one curated page per asset; each renders its image inline in Obsidian; the old `images/*.md` sidecars are not copied across.

**6. Generate the llm-wiki spine — `SCHEMA.md`, `index.md`, `log.md`** (templates in §6), plus the `_bases/*.base` dashboards.
- `SCHEMA.md` documents the frontmatter `type` field, folder semantics, and the wiki-embed convention.
- `index.md` is the unified Home landing page linking the knowledge layers *and* the life-admin Bases dashboards.
- `log.md` is seeded with the migration entries.
*Verify:* all three exist at the vault root and `index.md` links resolve in Obsidian.

**7. Commit and push the migrated vault.**
```bash
cd /opt/pkm-vault && git add -A && git commit -m "migrate: documents/ -> vault (assets + curated pages + spine + bases)"
GIT_CONFIG_GLOBAL=/dev/null git -c credential.helper='!gh auth git-credential' \
  push https://github.com/<owner>/pkm-vault.git main
```
*Verify:* GitHub shows the assets and pages (`gh repo view <owner>/pkm-vault --web`).

**8. Run the initial Hindsight full-rebuild reindex** so the index reflects the new vault, not the old chatter.
```bash
PKM_VAULT=/opt/pkm-vault python3 ~/.hermes/hindsight/reindex_from_vault.py --full-rebuild
```
*Verify:* a `hindsight_recall` for "motor control" returns the **`entities/program-motion-controller.md`** page reference (not a raw conversation fragment).

**9. Split the vault out of the config repo.** Stop tracking the proto-vault in `hermes-agent`: remove the `!documents/` un-ignore lines from its `.gitignore`, untrack `documents/`, and commit. The `db-snapshots/*.db` lines **stay** (config repo keeps backing up transient DBs).
```bash
git -C /workspaces/hermes-agent rm -r --cached documents
# edit .gitignore: drop the "!documents/" / "!documents/**" un-ignore block
git -C /workspaces/hermes-agent commit -am "migrate: vault moved to dedicated pkm-vault repo"
```
*Verify:* `git -C /workspaces/hermes-agent status` shows `documents/` untracked; the config repo no longer carries vault content.

**10. Retire the DB-as-truth model in `SOUL.md`.** Replace the line **"The markdown files are just convenient snapshots — the DB is the truth"** (find it with `grep -n "DB is the truth" ~/.hermes/SOUL.md` — do not rely on a fixed line number) and reframe the "How Memory Works" section so the **Obsidian vault is canonical**, Hindsight is a **rebuildable index over the vault**, and `state.db` is **transient working memory only**. Replace ingest/retrieve instructions with the §7 persona (write curated vault pages; return `obsidian://` deep links on retrieval).
*Verify:* `grep -n "DB is the truth" ~/.hermes/SOUL.md` returns nothing; the persona names the vault as source of truth.

**11. Decommission the proto-vault.** Once steps 1–10 verify, delete `documents/` from the working tree (history retained under the `pre-migration` tag).
```bash
rm -rf /workspaces/hermes-agent/documents
git -C /workspaces/hermes-agent commit -am "migrate: remove proto-vault working copy (history at tag pre-migration)"
```
*Verify:* on its next ingest the agent writes to `/opt/pkm-vault`, commits+pushes there, and Obsidian receives it within one pull interval — the migration is complete and the two-way loop is live.

**Post-migration:** run a full **lint** pass (§13; image-hygiene clears once sidecars are folded in, index completeness populated). Re-align the bespoke `devops/hermes-pkm-*` skills and `productivity/personal-knowledge-management` to "vault = source of truth, Hindsight = rebuildable index" (delta item #8).

---

## 17. Non-Goals and Open Questions

### Non-goals (out of scope for this system)

- **No multi-user / collaboration.** Single user only (`U0B5ZP301H8`). No sharing, permissions, or tenancy.
- **No bespoke web UI / dashboard app.** The "dashboard" is **Obsidian itself** (incl. Bases views); the conversational surfaces are Slack + MCP. No separate web front-end.
- **No chat-log-as-knowledge-base.** Conversation history is not a store. Only curated vault pages persist; chatter is not retained as knowledge (Hindsight is tuned to reflect the wiki, not the transcript).
- **No raw-source second store.** Uploaded originals live once in `raw/`; there is no parallel database copy treated as canonical.
- **No heavyweight project management.** Actions/TODOs with due/recurrence/priority — yes. Gantt charts, sprints, boards — no.
- **No email/SMS/other gateways for now.** Slack (primary) + MCP (secondary) only.
- **No use of the Claude Max OAuth credential by the agent**, ever (ban risk). Agent inference is pay-as-you-go Anthropic key only.

### Resolved during editing (recorded for traceability)

- **Vault name / path.** Standardised on Obsidian vault name `pkm-vault`, VPS path `/opt/pkm-vault`, env `PKM_VAULT`/`OBSIDIAN_VAULT_NAME=pkm-vault`. The earlier `hermes-vault` / `/home/hermes/vault` variant is dropped.
- **Model default.** One target: `claude-sonnet-4-6` (provider `anthropic`) for gateway, delegation, and cron — **verify it resolves (`hermes model`); dated `claude-sonnet-4-20250514` is the proven fallback**. Lighter models only via explicit per-call/delegate override. Resolves the Opus/Sonnet/Haiku drift; the cron 404 is fixed by moving off the bare `claude-sonnet-4`/`gemini-pro` ids to a resolvable one.
- **`auto_retain`.** Set to `false`; the index gains content only via deliberate retain-at-file-time and the reindex job.
- **`type` enum.** Unified to `entity|concept|comparison|query|summary|action|media|memory|inbox` (the Sync draft's `todo` maps to `action`).
- **Curator naming.** `auxiliary.curator` (LLM helper) and the top-level `curator` (skills curator) are both left as-is; vault reindex and lint are their own dedicated cron jobs.
- **DB snapshots.** Stay in the `hermes-agent` config repo as transient-working-memory backups; not moved to `pkm-vault`; Git LFS only if size strains the repo.
- **People sub-domain.** `people/` is kept as a first-class entity sub-domain; `index.md` gets a People section.

### Open questions (need confirmation against the live deployment)

1. **Obsidian vault registration.** Confirm the workstation (and phone) register the vault under exactly `pkm-vault`; if the folder name differs, the `vault=` parameter must match the *registered* name on each device or deep links fail there.
2. **`obsidian://` path form.** Verify the installed Obsidian URI handler accepts full URL-encoded vault-relative paths (with `%2F`) for both notes and non-md attachments; some versions accept basename-only for unique notes (which would break for assets, so full paths are specified here).
3. **Slack custom-scheme links.** Confirm the workspace's link settings don't strip `obsidian://`; if blocked, fall back to posting the raw URI as plain text plus the vault-relative path.
4. **Hindsight client surface — HARD PREREQUISITE, not a soft question (gates §9; see the ⚠️ callout there).** Verify against the *installed* package, before relying on any of it: (a) whether `hindsight_retain` / the `hindsight-client` Python class accept a first-class `reference` field that recall echoes back (the as-built docs attest only "store a fact with optional tags" — **no** `reference`); (b) the real Python class name and constructor (the spec's `Hindsight(mode=…, bank_id=…)` is unverified) vs the attested `hindsight-embed` CLI; (c) whether per-page `forget`/`forget_bank` and upsert-by-reference exist. **If `reference` is unsupported, the `SOURCE:` sentinel design in §9 is the shipping path** (no API dependency). The reindex script is marked illustrative until these are confirmed; the CLI/subprocess variant is the safe default.
5. **Hindsight daemon + embedding-model details.** Confirm the installed binary name (`hindsight-embed` vs other) and its subcommands (`db query`, `ui start`, `retain`/`forget` spellings), the local `local_embedded` UI port (8080 documented vs 8888/9999 for Docker), and the idle-timeout env var (`HINDSIGHT_IDLE_TIMEOUT`, default 300s). Also confirm the **embedding model** the Gemini `local_embedded` path actually uses — the spec pins **`text-embedding-004` at 3,072 dimensions** (per the `hermes-pkm-embeddings` skill; one curl sample uses the older `embedding-001`) — and **whether it is independently configurable** in `config.json` or fixed by the Hindsight build (`llm_model` selects only the *extraction* model, not embeddings). The free-tier 60k/month figure is anchored to `text-embedding-004`.
6. **Bases availability + date arithmetic (gates the dashboard layer — pick a v1).** First confirm the installed Obsidian even ships **Bases** (no repo skill documents it; the llm-wiki skill uses Dataview). Then validate `now() + "7 days"`, `file.mtime`, and `due_date < now() + …` against that Bases version. v1 target is Bases *if it validates*; **fallback is Dataview** (`TABLE … FROM "actions" WHERE …`, per §6) or status-only Bases with the cron briefing doing date math. Do not ship the Home dashboards on unvalidated Bases syntax — choose Bases-or-Dataview at build time.
7. **Hermes CLI verbs.** The spec standardises on the as-built **`hermes cron create "<schedule>" "<prompt>" --name … --model '{json}' --deliver …`** (positional schedule+prompt, JSON `--model`), per the `hermes-pkm-ops` skill. Confirm this exact form (and whether `create` vs `add` is the installed verb) and the Hindsight bank-management subcommands against the installed binary.
8. **Recurring-action reopen.** Confirm the desired behaviour for auto-reopening a completed recurring action and how the next `due_date` is computed (agent behaviour, §10).
9. **Unprivileged user.** Decide whether to create the dedicated `hermes` user during migration or defer (affects all `HERMES_HOME` paths in §15).
10. **Remote MCP only.** The local stdio MCP is fully specified (§15 Phase F.1: `hermes mcp serve`, the `~/.claude/settings.json` block, tools `pkm_ingest`/`pkm_search`/`pkm_todos`/`pkm_recent`) and needs no supervision (Claude Code spawns it per session). The remaining open item is **remote** exposure for claude.ai: whether to front `hermes mcp serve` HTTP/SSE with a Cloudflare Tunnel (§14 auth requirement) at all, and if so how it is supervised. Local-stdio is the v1 target; remote is optional/deferred.

---

## 18. Appendix

### A. Frontmatter quick reference

**Common (every page):** `title`, `type` (`entity|concept|comparison|query|summary|action|media|memory|inbox`), `created`, `updated`, `source` (`slack|mcp|web|manual|cron`), `tags`.

**Knowledge add-ons:** `sources` (list, required if any raw exists), `confidence` (`high|medium|low`), `contested` (`true`), `contradictions` (list), `aliases` (list).

**Life-admin add-ons:** `status`, `priority`, `due_date`, `recurrence`, `project` (action); `media_type`, `creator`, `url` (media); `people`, `location`, `memory_date` (memory/action).

**`raw/` block:** `source_url` (if any), `ingested`, `sha256` (digest of body below the closing `---`).

### B. Worked example — knowledge page that embeds an image

`entities/program-motion-controller.md` (migrated from the proto-vault `img_e4a408e064c4.png` + its sidecar — now one embed-and-describe page):

```markdown
---
title: Program Motion Controller (PMC)
type: entity
created: 2026-05-28
updated: 2026-05-30
source: slack
tags: [motor-control, embedded-systems, controls, product]
sources: [raw/assets/motor-control-diagram-e4a408.png]
confidence: medium
aliases: [PMC]
---

# Program Motion Controller (PMC)

Central coordinator in the motor-control stack: the PMC issues setpoints to the
[[drive-control-module]] (DCM), which drives the physical [[motor-rail-api]]. It
consumes **CS Demands** from the control system and exposes **CS Motor** state back.

![[motor-control-diagram-e4a408.png]]

The hand-drawn architecture above shows data flow PMC -> DCM -> Motor Rail/API, with
several components marked complete (red checkmarks) for progress tracking.
^[raw/assets/motor-control-diagram-e4a408.png]

## Key facts
- Role: central motion coordination.
- Talks to: [[drive-control-module]], [[motor-rail-api]].
- Inputs: CS Demands. Outputs: CS Motor state.

## Open questions
- Confidence is `medium`: single hand-drawn source; corroborate against a written spec.

## Related
- [[drive-control-module]] · [[ioc-network-architecture]]
```

The binary lives at `raw/assets/motor-control-diagram-e4a408.png`; there is **no** separate `img_*.md` description file.

### C. Worked example — Action / TODO page

`actions/fix-garden-fence.md`:

```markdown
---
title: Fix garden fence
type: action
created: 2026-05-30
updated: 2026-05-30
source: slack
tags: [task, home, errand]
status: todo
priority: 2 - High
due_date: 2026-06-07
recurrence: none
project: "[[home-maintenance]]"
people: ["[[people/jane-doe]]"]
---

# Fix garden fence

Two panels blown loose on the north side after the storm. Buy 2× featheredge
boards + galvanised nails; refit before the next forecast wind.

### Checklist
- [ ] Measure gap (approx 1.8 m)
- [ ] Buy boards + nails
- [ ] Refit and treat

### Notes
Linked to [[home-maintenance]]; coordinate with [[people/jane-doe]] for the weekend.
```

This page never needs an `index.md` entry — `_bases/actions.base` surfaces it under **Open Actions** / **Due Soon**, and the daily 07:00 Slack briefing reads it via the same frontmatter.

### D. Worked example — Media item

`media/ddia.md`:

```markdown
---
title: Designing Data-Intensive Applications
type: media
created: 2026-05-18
updated: 2026-05-30
source: slack
tags: [media, software, reference]
media_type: book
creator: Martin Kleppmann
url: https://dataintensive.net/
status: to_consume
priority: 3 - Medium
---

# Designing Data-Intensive Applications

Reference book on the architecture of data systems (replication, partitioning,
consistency, batch/stream processing). Captured for the to-consume backlog.

### Why
Background for the [[concepts/distributed-systems]] notes; covers the CAP
trade-offs filed at [[cap-theorem]].
```

Surfaced by `_bases/media.base` under **To Consume**; the daily summary may nudge it, and the weekly summary flags it if it goes cold (> 180 days).

### E. Example `obsidian://` deep links

```
# Note
obsidian://open?vault=pkm-vault&file=entities%2Fprogram-motion-controller.md
# Image attachment
obsidian://open?vault=pkm-vault&file=raw%2Fassets%2Fmotor-control-diagram-e4a408.png
# PDF attachment
obsidian://open?vault=pkm-vault&file=raw%2Fpapers%2Fcap-theorem-2002.pdf
# Path with a space
obsidian://open?vault=pkm-vault&file=concepts%2Fdistributed%20systems.md
```

### F. Config snippets

`~/.hermes/hindsight/config.json` (target). `llm_model` is the **extraction** model only (`gemini-2.5-flash`); embeddings use Gemini `text-embedding-004` / 3072-dim, configured by the build rather than this key (§9, verify per Open Question #5):

```json
{
  "mode": "local_embedded",
  "bank_id": "hermes",
  "llm_provider": "gemini",
  "llm_model": "gemini-2.5-flash",
  "auto_recall": true,
  "auto_retain": false,
  "memory_mode": "hybrid",
  "recall_budget": "mid",
  "recall_prefetch_method": "recall",
  "recall_max_tokens": 4096,
  "retain_async": true,
  "retain_context": "facts already filed into the Obsidian vault by Hermes"
}
```

`config.yaml` (target highlights):

```yaml
model:
  default: claude-sonnet-4-6      # verify with `hermes model`; fallback: claude-sonnet-4-20250514
  provider: anthropic
delegation:
  model: claude-sonnet-4-6        # same id as default (and same fallback)
  provider: anthropic
  max_concurrent_children: 2
  max_spawn_depth: 1
memory:
  provider: hindsight
  memory_enabled: true
slack:
  require_mention: true
approvals:
  mode: auto
security:
  tirith_enabled: true
  redact_secrets: true
  allow_private_urls: false
timezone: Europe/London
web:
  search_backend: exa
  extract_backend: firecrawl
```

`pkm-vault/.gitignore`:

```gitignore
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache
.trash/
.DS_Store
*.tmp
.obsidian/plugins/obsidian-git/.gitignore
```

One-shot vault scaffold (VPS):

```bash
VAULT=/opt/pkm-vault
mkdir -p "$VAULT"/{raw/{articles,papers,transcripts,assets},entities,concepts,comparisons,queries,actions,media,memories,people,_bases,_meta,_archive,inbox,.obsidian}
: > "$VAULT/log.md"   # then write the SCHEMA.md / index.md / log.md / _bases templates from §6
```
