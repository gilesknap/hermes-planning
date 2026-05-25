# Hermes PKM Agent -- Specification

## Overview

A Personal Knowledge Management agent running on a VPS via Hermes Agent, serving as a
persistent second brain. Content is ingested primarily through Slack (conversational),
with the same capabilities exposed via MCP for use within Claude Code and claude.ai
desktop sessions. The agent auto-categorizes everything -- the user just throws content
at it and retrieves it with natural language.

## User Profile

- **Timezone:** Europe/London (GMT/BST)
- **Subscription:** Claude MAX 100 (API token strategy TBD)

---

## Core Capabilities

### 1. Ingestion

The agent accepts content in any of these forms and auto-categorizes it:

| Content Type | Input Method | Processing |
|-------------|-------------|------------|
| **Web pages** | Paste URL in Slack or MCP | Fetch page, generate summary + tags, store URL + summary |
| **Markdown** | Paste or attach in Slack | Parse, summarize, tag, store |
| **PDFs** | Attach in Slack | Extract text (OCR for scanned docs), summarize, tag, store file + extracted text |
| **Images** | Attach in Slack | AI description + OCR where applicable, tag, store image + description |
| **Snippets** | Just type/paste into Slack | Auto-detect type (note, code, idea, quote, etc.), categorize, store |
| **TODOs** | Natural language in Slack | Parse due date, recurrence, priority; create structured reminder |

**Auto-categorization:** The agent determines content type and applies tags/categories
without requiring the user to specify anything. The user just sends content and the
agent files it.

### 2. Storage & Indexing

- All content stored locally on the VPS (Hermes' built-in SQLite with FTS5)
- Each item gets: unique ID, timestamp, auto-generated tags, AI summary, source reference
- Full-text search across all stored content
- Images and PDFs stored as files with searchable text descriptions
- Web pages stored as URL + summary (no local cache of full page content -- trust the original stays up)

### 3. Retrieval

**Natural language search via Slack and MCP:**
- "Find that article about X"
- "What did I save about Y last week?"
- "Show me all my Python code snippets"
- "What TODOs are due this week?"

**Original document access (via Slack):**
- Quick retrieval of full original documents: URLs opened directly, PDFs/images sent as attachments
- MCP retrieval returns text content and metadata (file attachments are a Slack strength, harder via MCP)

### 4. TODO & Reminders

- **Due dates:** absolute and relative ("next Tuesday", "in 3 days")
- **Recurrence:** daily, weekly, monthly, custom patterns
- **Priority:** high / medium / low
- **Categories/tags:** auto-assigned, can be overridden
- **Status tracking:** pending, in progress, done, deferred
- Overdue items highlighted in daily summary

### 5. Proactive Summaries

**Daily Summary -- 7:00 AM UK time (via Slack):**
- TODOs due today and overdue items
- Upcoming deadlines (next 3 days)
- Brief recap of what was ingested yesterday
- Any items flagged for follow-up

**Weekly Summary -- Monday 7:00 AM UK time (via Slack):**
- Week-in-review: what was ingested, key themes
- TODO completion rate and outstanding items
- Upcoming week's deadlines
- Suggested items to review or archive

---

## Access Channels

### Slack (Primary)
- Conversational interface in a dedicated channel or DM
- Send any content to ingest (URLs, files, images, text)
- Natural language queries to retrieve
- Receives daily/weekly summaries
- Full document retrieval (attachments, links)

### MCP Server (Secondary)
- Exposed as an MCP server for Claude Code and claude.ai desktop
- Same ingestion and retrieval capabilities as Slack
- Best for: searching knowledge base during work sessions, adding technical notes
- Limitation: file/image retrieval less practical than Slack (text-based responses)

---

## Architecture (High Level)

```
User
  |
  +-- Slack ---------> Hermes Agent (VPS) ----> SQLite + FTS5 (knowledge store)
  |                         |                        |
  +-- MCP (Claude Code) ---+                   File storage (PDFs, images)
  |                         |
  +-- MCP (claude.ai) -----+
                            |
                       Claude API (LLM for summarization, categorization, search)
                            |
                       Cron scheduler (daily/weekly summaries)
```

---

## Backup & Recovery

**Goal:** Reasonably quick recovery if the VPS is lost. Same-day restoration acceptable.

### Strategy: Single Private GitHub Repo

Everything backed up to one private GitHub repo on a 6-hour cron cycle.

**Repo structure:**
```
hermes-backup/
  config/          # ~/.hermes config, skills, memory markdown (text, git-friendly)
  data/
    db/            # SQLite .backup snapshot (safe copy, not live DB file)
    files/         # Ingested PDFs, images (<1GB total expected)
```

**Backup process (every 6 hours via Hermes cron or system cron):**
1. Run `sqlite3 ~/.hermes/knowledge.db ".backup /tmp/hermes-backup.db"` (safe online backup)
2. Copy backup DB + ingested files to repo working tree
3. `git add -A && git commit -m "backup $(date -u +%Y-%m-%dT%H:%MZ)" && git push`

**Recovery process:**
1. Provision new VPS, install Hermes Agent
2. Clone backup repo
3. Restore config to `~/.hermes/`, restore DB and files to data directories
4. Reconfigure Slack/MCP connections, restart agent

**Capacity:** GitHub repos support up to 5GB (with LFS up to more). At <1GB expected
volume this is well within limits. Git history provides point-in-time snapshots for free.

**What's NOT backed up:**
- Active session state (ephemeral, acceptable to lose)
- Slack message history (lives in Slack)

---

## Non-Goals (for now)

- No collaborative/multi-user features -- single user only
- No web UI dashboard (Slack + MCP are the interfaces)
- No local caching of full web pages (just URL + summary)
- No complex project management (TODOs yes, Gantt charts no)
- No email integration (Slack and MCP only)

---

## Open Questions

1. **API token strategy** -- to be decided separately (MAX Agent SDK credit vs pay-as-you-go vs hybrid)
2. **Storage limits** -- how much disk on the VPS? Affects PDF/image retention policy
3. **Slack workspace** -- existing workspace or new dedicated one?
4. **MCP transport** -- stdio for local Claude Code, SSE/streamable-HTTP for claude.ai remote access?
