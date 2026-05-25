# API Token Options -- Hermes PKM Agent

Companion to `API-TOKENS.md`. This document explores the alternatives for each token
category so the trade-offs are visible and decisions can be revisited.

---

## 1. LLM Provider Options

The Hermes Agent needs an LLM for summarization, categorization, conversation, and
search. Any OpenAI-compatible API works.

### Option A: Anthropic Direct (Pay-as-you-go) -- Selected

Use `ANTHROPIC_API_KEY` from console.anthropic.com with standard billing.

| Model | Input/1M tokens | Output/1M tokens | Context | Notes |
|-------|----------------|------------------|---------|-------|
| Haiku 4.5 | $0.80 | $4.00 | 200K | Fast, cheap, good for routine tasks |
| Sonnet 4.6 | $3.00 | $15.00 | 200K | Best cost/capability balance |
| Opus 4.7 | $15.00 | $75.00 | 1M | Most capable, expensive |

**Estimated monthly cost for PKM (50-100 calls/day):**
- Haiku-only: ~$5-10/month
- Sonnet-only: ~$15-30/month
- Hybrid (Haiku for ingest, Sonnet for search/summaries): ~$8-15/month

**Pros:** Direct, lowest latency, full feature support (caching, extended thinking), Hermes default provider.
**Cons:** Requires billing setup, no free tier.

### Option B: Anthropic via MAX Agent SDK Credit (June 15+)

After June 15, 2026, MAX 100 subscribers get $100/month in Agent SDK credit consumed
at standard API rates.

**Pros:** Effectively free (already paying for MAX), no separate billing.
**Cons:** Not available until June 15. $100/month cap -- if exceeded, either requests stop
or overage is billed. Credit doesn't roll over. Unclear if Hermes can authenticate via
Agent SDK flow (may require standard API key anyway, with billing just routed differently).

**Open question:** Does the Agent SDK credit work with a standard `ANTHROPIC_API_KEY`, or
does it require Agent SDK authentication? If the latter, Hermes would need to support
that auth flow. Worth investigating once June 15 arrives.

### Option C: OpenRouter

Route through OpenRouter to access Claude (or any of 200+ models) with a single key.

| Aspect | Detail |
|--------|--------|
| Markup | ~10-20% over direct provider pricing |
| Free models | Some open-source models available at $0 |
| Fallback | Auto-failover to alternative models if primary is down |

**Pros:** Single key for multiple providers, model fallback, can mix Claude with cheaper
models for different tasks.
**Cons:** Added latency (extra hop), markup on pricing, another dependency.

### Option D: Self-Hosted / Open Source Models

Run a local model via Ollama or vLLM on the VPS.

| Model | VRAM Needed | Quality vs Claude |
|-------|-------------|-------------------|
| Llama 3.3 70B | ~40GB | Good for simple tasks, weaker on nuance |
| Qwen 2.5 72B | ~40GB | Competitive on coding/structured tasks |
| Mistral Small | ~16GB | Decent for categorization |
| Phi-4 14B | ~10GB | Lightweight, limited capability |

**Pros:** No API cost, full privacy, no rate limits.
**Cons:** Requires GPU (expensive VPS upgrade), significantly lower quality than Claude
for nuanced tasks like summarization and natural language search, maintenance burden.
Not practical unless the VPS has a GPU.

### Option E: Google Gemini (LLM, not just embeddings)

Gemini also has a free tier for generative models.

| Model | Free Tier | Paid Rate |
|-------|-----------|-----------|
| Gemini 2.0 Flash | 15 RPM, 1M TPM | $0.10/$0.40 per 1M tokens |
| Gemini 2.5 Pro | 5 RPM, 250K TPM | $1.25-2.50/$10-15 per 1M tokens |

**Pros:** Free tier could cover light PKM use, very cheap paid tier.
**Cons:** Lower quality than Claude for nuanced summarization (subjective), free tier
rate limits tight for bursty workloads, Google data usage policy on free tier.

### Hybrid Strategies

Hermes supports model switching, so you could:
- **Cheap model for ingest** (Haiku or Gemini Flash for tagging/categorizing) + **capable model for retrieval** (Sonnet for answering questions and generating summaries)
- **Free tier for routine** (Gemini Flash) + **Claude for complex** (Sonnet via API key)
- **Agent SDK credit for baseline** + **pay-as-you-go overflow** after June 15

---

## 2. Embedding Provider Options

Embeddings power semantic search -- converting text into vectors so "find articles about
distributed systems" matches content that never uses that exact phrase.

### Option A: Google Gemini (Free Tier) -- Selected

| Model | Dimensions | MTEB Score | Free Tier | Cost (Paid) |
|-------|-----------|------------|-----------|-------------|
| gemini-embedding-001 | up to 3072 | 68.32 | ~1,000 RPD | N/A (free only) |
| Gemini Embedding 2 | up to 3072 | 69.9 | Free (preview) | TBD |

**Pros:** Free, high quality, large context (8K tokens), flexible dimensions.
**Cons:** No SLA, data may be used for training, possible quota glitches, model
deprecation risk requires re-embedding.

**Multimodal bonus:** Gemini Embedding 2 can embed images directly -- could enable
semantic image search without relying on text descriptions. Worth evaluating at GA.

### Option B: OpenAI

| Model | Dimensions | MTEB Score | Cost/1M tokens |
|-------|-----------|------------|----------------|
| text-embedding-3-small | 1536 | ~62 | $0.02 |
| text-embedding-3-large | 3072 | ~64 | $0.13 |

**Pros:** Rock-solid reliability, widely supported, very cheap.
**Cons:** Lower quality than Gemini on benchmarks, not free, requires OpenAI account.

### Option C: Voyage AI

| Model | Dimensions | MTEB Score | Cost/1M tokens |
|-------|-----------|------------|----------------|
| voyage-3-lite | 512 | ~63 | $0.02 |
| voyage-3 | 1024 | ~67 | $0.06 |
| voyage-3-large | 1024 | ~68 | $0.18 |

**Pros:** Top-tier retrieval quality, recommended by Anthropic, 32K context window.
**Cons:** Not free, smaller company (longevity risk?), requires separate account.

### Option D: Cohere

| Model | Dimensions | Cost/1M tokens |
|-------|-----------|----------------|
| embed-v4 | 1024 | $0.10 |
| embed-v4 (search) | 1024 | $0.10 |

**Pros:** Good multilingual support, has a limited free trial tier.
**Cons:** More expensive than OpenAI, smaller ecosystem.

### Option E: Self-Hosted (Ollama / Sentence Transformers)

Run embedding models locally on the VPS CPU.

| Model | Dimensions | Quality | CPU Speed |
|-------|-----------|---------|-----------|
| nomic-embed-text | 768 | Good | ~50ms/embed |
| all-MiniLM-L6-v2 | 384 | Decent | ~20ms/embed |
| mxbai-embed-large | 1024 | Good | ~100ms/embed |

**Pros:** Free, private, no rate limits, no external dependency.
**Cons:** Lower quality than cloud models, uses VPS CPU/RAM, need to manage the service.

### Key Decision Factors

| Factor | Gemini Free | OpenAI | Voyage | Self-Hosted |
|--------|------------|--------|--------|-------------|
| Cost | Free | ~$0.50/mo | ~$1.50/mo | Free |
| Quality (MTEB) | 68.3 | ~62 | ~68 | ~55-63 |
| Reliability | No SLA | High | High | Self-managed |
| Privacy | Data may be used | Standard ToS | Standard ToS | Full privacy |
| Vendor lock-in | Low (re-embed) | Low | Low | None |

**Important note on switching:** Changing embedding providers means re-embedding all
stored content, since vectors from different models are incompatible. Choose once and
stick with it, or store source text alongside embeddings so re-embedding is possible.

---

## 3. Slack Integration Options

### Option A: Socket Mode (Recommended for VPS) -- Selected

The agent opens an outbound WebSocket connection to Slack. No public URL needed.

**Tokens:** Bot token (`xoxb-...`) + App-level token (`xapp-...`)

**Pros:** No firewall/DNS/SSL config, works behind NAT, simple.
**Cons:** Limited to one workspace per app, slightly higher latency than Events API.

### Option B: Events API (HTTP Webhooks)

Slack sends HTTP POST requests to a public endpoint on the VPS.

**Tokens:** Bot token (`xoxb-...`) + Signing secret

**Pros:** Lower latency, supports multiple workspaces, more scalable.
**Cons:** Requires public URL, SSL certificate, firewall rules, reverse proxy (nginx).
Overkill for single-user personal PKM.

### Option C: Slack Bolt (Framework)

Use Slack's official Bolt framework (Python) instead of Hermes gateway.

**Pros:** Official SDK, well-documented, handles auth flows.
**Cons:** Hermes already has a Slack gateway adapter -- using Bolt would mean bypassing
Hermes' built-in integration. Only relevant if the Hermes adapter is insufficient.

### Workspace Setup

| Approach | Pros | Cons |
|----------|------|------|
| Existing workspace | No setup, already there | PKM noise mixed with other channels |
| Dedicated workspace | Clean separation, focused | Another workspace to manage |
| DM-only in existing | Minimal, private | Limited to DM, no channel organization |

---

## 4. GitHub PAT Options for Backup

### Option A: Fine-Grained PAT (Recommended) -- Selected

Scoped to a single private repo with minimal permissions.

**Pros:** Least privilege, clear audit trail, repo-scoped.
**Cons:** Fine-grained PATs are newer, some git tooling has edge cases.

### Option B: Classic PAT with `repo` Scope

Traditional PAT with full repo access.

**Pros:** Simple, universally supported.
**Cons:** Grants access to ALL repos (private and public), overly broad for a backup cron.

### Option C: Deploy Key

SSH key added to a single repo as a deploy key with write access.

**Pros:** Repo-scoped like fine-grained PAT, no GitHub account-wide access.
**Cons:** SSH-based (our git config uses HTTPS via gh), one key per repo, manage SSH agent.

### Option D: GitHub App (Installation Token)

Create a GitHub App, install it on the backup repo, generate short-lived tokens.

**Pros:** Short-lived tokens (1 hour), fine-grained permissions, best security.
**Cons:** Complex setup (app registration, private key, token refresh logic), overkill for
a personal backup cron.

### PAT Security Comparison

| Method | Scope | Token Lifetime | Setup Complexity |
|--------|-------|---------------|-----------------|
| Fine-grained PAT | Single repo | Configurable (30-365 days) | Low |
| Classic PAT | All repos | Configurable | Low |
| Deploy key | Single repo | No expiry | Medium |
| GitHub App | Per-installation | 1 hour (auto-refresh) | High |

---

## Decision Summary

| Category | Selected | Rationale |
|----------|----------|-----------|
| LLM | Anthropic direct (pay-as-you-go) | Best quality, Hermes default, re-evaluate at June 15 |
| Embeddings | Gemini free tier | Free, high quality, sufficient quota for PKM |
| Slack | Socket Mode | Simple, no public URL needed, free |
| GitHub backup | Fine-grained PAT | Least privilege, repo-scoped, easy setup |
