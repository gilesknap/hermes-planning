# API Tokens -- Hermes PKM Agent

Four tokens are needed to run the full PKM agent setup.

---

## 1. Anthropic API Key (LLM)

**Purpose:** All LLM calls -- summarization, auto-categorization, natural language search,
conversation, daily/weekly summary generation.

**Provider:** Anthropic (console.anthropic.com)

**Auth type:** Pay-as-you-go API key

**Setup:**
1. Create account at [console.anthropic.com](https://console.anthropic.com)
2. Add billing (credit card)
3. Generate API key
4. Set `ANTHROPIC_API_KEY` in `~/.hermes/.env`

**Cost considerations:**
- Pricing depends on model choice. For a PKM with light-moderate usage:
  - **Sonnet 4.6** ($3/$15 per 1M input/output tokens) -- good balance of cost and capability
  - **Haiku 4.5** ($0.80/$4 per 1M tokens) -- cheapest, fine for routine categorization/tagging
  - **Opus 4.7** ($15/$75 per 1M tokens) -- overkill for most PKM tasks
- A hybrid approach is possible: Haiku for ingestion/tagging, Sonnet for search/summaries
- Estimate for light PKM use (50-100 calls/day): roughly $5-30/month depending on model

**Future option:** On June 15, 2026, the MAX 100 subscription gains a $100/month Agent SDK
credit pool. This could supplement or replace pay-as-you-go depending on usage volume.
The Agent SDK credit uses standard API rates, so $100 goes the same distance. Worth
re-evaluating once it's live -- if PKM usage stays under $100/month, could eliminate
the separate API bill entirely.

**Hermes config:** Set the model in `~/.hermes/config.yaml` or switch at runtime with
`hermes model`.

---

## 2. Google Gemini API Key (Embeddings)

**Purpose:** Generating vector embeddings for semantic search and indexing of all
ingested content (web page summaries, PDFs, images, snippets, TODOs).

**Provider:** Google AI Studio (free tier)

**Model:** `gemini-embedding-001`
- 3072 dimensions (configurable down to 128 via Matryoshka)
- 8,192 token context window
- MTEB score 68.32 (outperforms OpenAI text-embedding-3-small)

**Setup:**
1. Get API key from [aistudio.google.com](https://aistudio.google.com/apikey)
2. Set in `~/.hermes/.env` (exact var name depends on Hermes embedding config)

**Free tier limits:**
- ~1,000 requests/day
- 10M tokens/minute
- No credit card required

**Expected usage:** 20-100 calls/day for ingestion + search queries. Well within free limits.

**Trade-offs:**
- Google may use free-tier data for model training (acceptable for personal PKM)
- No SLA -- occasional quota glitches reported
- If model is deprecated, all content needs re-embedding (mitigate by storing source text)

**Future consideration:** Gemini Embedding 2 (currently preview, free) supports multimodal
embedding (text + image + audio). Could enable direct image semantic search without
relying solely on text descriptions. Worth evaluating once it reaches GA.

---

## 3. Slack Bot Token

**Purpose:** Primary user interface -- ingesting content, retrieving knowledge,
receiving daily/weekly summaries.

**Provider:** Slack (api.slack.com)

**Tokens needed:**
- **Bot Token** (`xoxb-...`) -- for sending/receiving messages, file uploads/downloads
- **App-Level Token** (`xapp-...`) -- for Socket Mode (recommended for VPS, no public URL needed)

**Required OAuth scopes (minimum):**
```
chat:write          # Send messages
files:read          # Read uploaded files (PDFs, images)
files:write         # Send files back (retrieval)
im:history          # Read DM history
im:read             # Access DM channel info
im:write            # Open DMs
```

**Setup:**
1. Create a Slack App at [api.slack.com/apps](https://api.slack.com/apps)
2. Enable Socket Mode (Settings > Socket Mode) -- generates app-level token
3. Add Bot Token Scopes under OAuth & Permissions
4. Install to workspace -- generates bot token
5. Set both tokens in `~/.hermes/.env`

**Socket Mode vs webhooks:** Socket Mode is recommended for a VPS setup because the
agent connects outbound to Slack (no need for a public URL, SSL cert, or ingress rules).
The Hermes Slack gateway adapter should support this.

**Cost:** Free for a single-workspace personal app.

---

## 4. GitHub PAT (Backup Repo)

**Purpose:** Pushing automated backups (config + SQLite snapshot + ingested files) to a
private GitHub repo every 6 hours.

**Provider:** GitHub (github.com/settings/tokens)

**Type:** Fine-grained Personal Access Token

**Scope:** Restrict to the single private backup repo with these permissions:
- **Contents:** Read and write (push commits)
- **Metadata:** Read-only (required)

**Setup:**
1. Create the private backup repo: `gh repo create hermes-backup --private`
2. Generate fine-grained PAT at github.com/settings/tokens scoped to that repo only
3. Configure on VPS: `gh auth login --with-token <<< "github_pat_..."`
4. Or set directly in git credential store for the backup repo

**Important:** This is a separate PAT from the one used for the public planning repo
(`gilesknap/hermes-planning`). Keep them scoped independently -- the backup PAT should
only have access to the private backup repo.

**Cost:** Free.

**Rotation:** GitHub fine-grained PATs can be set to expire. Consider 90-day expiry with
a reminder to rotate (the agent could even remind you via the daily summary).

---

## Summary

| Token | Provider | Cost | Rotation |
|-------|----------|------|----------|
| Anthropic API key | console.anthropic.com | ~$5-30/month (model-dependent) | Manual, no expiry |
| Gemini embedding key | aistudio.google.com | Free | No expiry |
| Slack bot + app tokens | api.slack.com | Free | No expiry (app-level) |
| GitHub PAT (backup) | github.com | Free | Set 90-day expiry |

**Total recurring cost:** ~$5-30/month (Anthropic API only), potentially $0 after June 15
if Agent SDK credit covers usage.

---

## Security Notes

- All tokens stored in `~/.hermes/.env` (not in config.yaml or git)
- The `.env` file should be `chmod 600`
- Hermes strips API keys from child process environments by default
- Backup repo is private -- tokens never appear in the public planning repo
- Consider using the VPS's secret management if available (e.g., systemd credentials)
