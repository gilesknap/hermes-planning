# Hermes PKM Agent -- Setup Runbook

Step-by-step guide to setting up the Hermes PKM Agent on this VPS. Designed to be
followed by an AI agent (Sonnet 4.7 via Hermes) or a human. Each section has
verification steps -- don't proceed until the current section passes.

**VPS baseline (as of 2026-05-25):**
- Ubuntu 26.04 LTS, 2 cores, 8GB RAM, 94GB disk
- Python 3.14 (system) -- exceeds Hermes' 3.11+ minimum
- git 2.53
- Missing: uv, Node.js, git-lfs, sqlite3, docker, pip, npm

**Reference docs:**
- `PKM-AGENT-SPEC.md` -- what we're building
- `API-TOKENS.md` -- token setup details
- `BACKUP-OPTIONS.md` -- backup strategy

---

## Phase 1: VPS Prerequisites

### 1.1 System packages

```bash
apt update && apt install -y \
  sqlite3 \
  git-lfs \
  jq \
  build-essential \
  libffi-dev \
  libssl-dev
```

**Verify:**
```bash
sqlite3 --version    # should show 3.x
git lfs version      # should show git-lfs/3.x
```

### 1.2 Install uv (Python package manager)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc   # or restart shell
```

**Verify:**
```bash
uv --version   # should show 0.x or 1.x
```

### 1.3 Install Node.js 20+

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
```

**Verify:**
```bash
node --version   # should show v22.x
npm --version    # should show 10.x+
```

### 1.4 Initialize git-lfs

```bash
git lfs install
```

---

## Phase 2: Install Hermes Agent

### 2.1 Clone the repository

```bash
cd /opt
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
```

### 2.2 Create Python virtual environment

Hermes requires Python 3.11+. The system has 3.14 which exceeds this. If there
are compatibility issues with 3.14 (bleeding edge), uv can install an older
Python automatically:

```bash
# Use system Python (3.14)
uv venv venv --python 3.14

# Fallback if 3.14 causes issues (uv downloads it automatically):
# uv venv venv --python 3.12
```

### 2.3 Install Hermes and dependencies

```bash
export VIRTUAL_ENV="/opt/hermes-agent/venv"
uv pip install -e ".[all,dev]"
npm install   # for TUI and browser tools
```

### 2.4 Add to PATH

```bash
echo 'export PATH="/opt/hermes-agent/venv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

**Verify:**
```bash
hermes --version    # should show v0.14.x
hermes doctor       # should pass basic checks
```

---

## Phase 3: API Token Configuration

### 3.1 Create Hermes home directory

```bash
mkdir -p ~/.hermes
cp /opt/hermes-agent/cli-config.yaml.example ~/.hermes/config.yaml
cp /opt/hermes-agent/.env.example ~/.hermes/.env
chmod 600 ~/.hermes/.env
```

### 3.2 Anthropic API Key (LLM)

1. Go to https://console.anthropic.com
2. Create account / sign in, add billing
3. Go to API Keys, create a new key
4. Edit `~/.hermes/.env`:

```bash
ANTHROPIC_API_KEY=sk-ant-...
```

**Model selection:** Edit `~/.hermes/config.yaml` and set the default model.
For PKM use, Sonnet 4.6 is the recommended balance of cost and capability:

```yaml
model: anthropic/claude-sonnet-4-6
```

For cost savings on routine ingestion tasks, consider configuring Haiku as a
secondary model (see Hermes docs for model routing/fallback configuration).

**Verify:**
```bash
hermes --test-api   # or start a session and send a test message
```

### 3.3 Google Gemini API Key (Embeddings)

1. Go to https://aistudio.google.com/apikey
2. Create a new API key (no billing required)
3. Edit `~/.hermes/.env`:

```bash
GOOGLE_GEMINI_API_KEY=AIza...
```

**Embedding model configuration:** This step depends on how Hermes configures its
embedding pipeline. Check the following locations in the Hermes source:

```bash
# Find embedding-related config
grep -r "embed" /opt/hermes-agent/cli-config.yaml.example
grep -r "embedding" /opt/hermes-agent/plugins/
ls /opt/hermes-agent/plugins/memory/
```

The embedding model should be set to `gemini-embedding-001` with 3072 dimensions.
The exact config key will be in `config.yaml` or the memory plugin configuration.
Look for settings like:

```yaml
# Likely in config.yaml or a memory plugin config:
embedding_model: gemini-embedding-001
embedding_dimensions: 3072
embedding_provider: gemini
```

**Verify:** Ingest a test document and confirm that semantic search returns it
(not just keyword matching).

### 3.4 Slack Bot Token

#### Create the Slack App

1. Go to https://api.slack.com/apps
2. Click "Create New App" > "From scratch"
3. Name: `Hermes PKM` (or your preference)
4. Pick your workspace
5. Click "Create App"

#### Enable Socket Mode

6. Left sidebar: Settings > Socket Mode
7. Toggle "Enable Socket Mode" to ON
8. It will prompt you to create an app-level token
9. Name: `hermes-socket`, scope: `connections:write`
10. Copy the `xapp-...` token

#### Configure Bot Scopes

11. Left sidebar: Features > OAuth & Permissions
12. Scroll to "Scopes" > "Bot Token Scopes"
13. Add these scopes:

```
chat:write          # Send messages
files:read          # Read uploaded files (PDFs, images)
files:write         # Send files back (retrieval)
im:history          # Read DM history
im:read             # Access DM channel info
im:write            # Open DMs
```

#### Enable Events

14. Left sidebar: Features > Event Subscriptions
15. Toggle "Enable Events" to ON (Socket Mode handles the URL)
16. Under "Subscribe to bot events", add:

```
message.im          # DM messages (primary interaction)
file_shared          # File uploads (PDF, image ingestion)
```

#### Install to Workspace

17. Left sidebar: Settings > Install App
18. Click "Install to Workspace" and authorize
19. Copy the Bot User OAuth Token (`xoxb-...`)

#### Save Tokens

Edit `~/.hermes/.env`:

```bash
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
```

**Verify:**
```bash
# Test the bot token
curl -s -H "Authorization: Bearer xoxb-YOUR-TOKEN" \
  https://slack.com/api/auth.test | jq .
# Should show "ok": true and your bot's user info
```

---

## Phase 4: Configure Hermes as a PKM Agent

### 4.1 System Prompt / Personality

Edit `~/.hermes/config.yaml` to set the system prompt that makes Hermes behave as
a PKM agent. Look for the `system_prompt` or `personality` key:

```yaml
system_prompt: |
  You are a Personal Knowledge Management assistant. Your role is to help the user
  capture, organize, and retrieve information.

  When the user sends you content:
  1. Determine the content type (URL/webpage, note, code snippet, idea, quote, TODO, image, PDF)
  2. Auto-categorize with relevant tags
  3. Generate a concise summary
  4. Store it with full-text search indexing
  5. Confirm what you stored with a brief acknowledgment

  When the user asks a question or searches:
  1. Search the knowledge base using semantic search and full-text search
  2. Return relevant results with context
  3. For URLs, provide the original link for quick access
  4. For files (PDFs, images), offer to send them as attachments

  For TODOs:
  - Parse natural language for due dates, recurrence, and priority
  - Support: daily, weekly, monthly, custom recurrence
  - Priority levels: high, medium, low
  - Track status: pending, in progress, done, deferred

  Be concise in responses. Acknowledge ingestion in 1-2 lines. Provide search
  results clearly with source references.
```

### 4.2 Enable Required Toolsets

Check `~/.hermes/config.yaml` for toolset configuration. The PKM agent needs:

```yaml
toolsets:
  - memory          # persistent knowledge storage
  - file_ops        # read/write files (PDFs, images)
  - web_extract     # fetch and extract web page content
  - web_search      # search capabilities
  - cron            # scheduled summaries
  # Add others as needed based on available toolsets
```

To see all available toolsets:
```bash
grep -r "toolset" /opt/hermes-agent/toolsets.py | head -40
```

### 4.3 Configure the Slack Gateway

Check how Hermes configures gateway platforms:

```bash
ls /opt/hermes-agent/gateway/platforms/
cat /opt/hermes-agent/gateway/platforms/slack*.py | head -50
```

Configuration is likely in `config.yaml`:

```yaml
gateway:
  platform: slack
  mode: socket    # Socket Mode, not webhooks
```

Or it may use command-line flags:
```bash
hermes --gateway slack
```

### 4.4 Memory / Embedding Pipeline

Hermes has multiple memory plugins. Check which ones are available and configure
the one that supports external embeddings:

```bash
ls /opt/hermes-agent/plugins/memory/
```

Configure the selected memory plugin to use Gemini embeddings. This may involve:
- Setting the embedding provider in config.yaml
- Installing a Gemini embedding integration if not built-in
- Or configuring a custom embedding endpoint

If Hermes doesn't natively support Gemini embeddings, you may need to:
1. Check if there's an OpenAI-compatible embedding proxy option (Gemini's API
   isn't OpenAI-compatible for embeddings)
2. Or use the `litellm` proxy which can translate Gemini embedding calls to
   OpenAI format
3. Or write a small adapter skill

**Investigate:**
```bash
grep -ri "embed" /opt/hermes-agent/plugins/memory/ --include="*.py" -l
grep -ri "embedding_provider\|embedding_model\|embed_model" /opt/hermes-agent/ --include="*.py" -l | head -20
```

---

## Phase 5: MCP Server Setup

### 5.1 Understand Hermes MCP Support

Hermes has MCP integration. Check what's available:

```bash
grep -ri "mcp" /opt/hermes-agent/ --include="*.py" -l | head -20
ls /opt/hermes-agent/tools/mcp* 2>/dev/null
```

There are two directions for MCP:
- **Hermes as MCP client** (using external MCP servers as tools) -- built-in
- **Hermes as MCP server** (exposing PKM capabilities to Claude Code/claude.ai) -- may need custom setup

### 5.2 Option A: Hermes Has Built-in MCP Server Mode

If Hermes can expose itself as an MCP server:

```bash
hermes --mcp-server   # or similar flag
```

Configure in Claude Code's settings (`.claude/settings.json`):
```json
{
  "mcpServers": {
    "hermes-pkm": {
      "command": "/opt/hermes-agent/venv/bin/hermes",
      "args": ["--mcp-server"]
    }
  }
}
```

### 5.3 Option B: Custom MCP Server Wrapper

If Hermes doesn't have native MCP server mode, write a thin MCP server that
calls Hermes' Python API:

```bash
# Check if Hermes exposes a Python API for search/ingest
grep -r "class AIAgent" /opt/hermes-agent/agent/ --include="*.py" -l
grep -r "def search\|def ingest\|def store" /opt/hermes-agent/agent/ --include="*.py" | head -20
```

A minimal MCP server would expose these tools:
- `pkm_ingest` -- store content (text, URL, snippet)
- `pkm_search` -- search the knowledge base
- `pkm_todos` -- list/create/update TODOs
- `pkm_recent` -- show recently ingested items

### 5.4 Remote MCP for claude.ai

For claude.ai desktop access, the MCP server needs to be reachable remotely.
Options:
- **SSH tunnel:** `ssh -L 3000:localhost:3000 vps` (manual, per-session)
- **Streamable HTTP:** Expose MCP over HTTP with auth (needs reverse proxy + auth token)
- **Cloudflare Tunnel:** Free, no public IP needed, encrypted

**Investigate claude.ai MCP support:**
The claude.ai desktop app supports remote MCP servers. Check current docs for
the transport format (likely streamable-HTTP or SSE).

---

## Phase 6: Scheduled Summaries

### 6.1 Daily Summary (7:00 AM UK time)

Use Hermes' built-in cron scheduler:

```bash
hermes cron add \
  --schedule "0 7 * * *" \
  --timezone "Europe/London" \
  --name "daily-summary" \
  --prompt "Generate the daily PKM summary. Include: TODOs due today and overdue items, upcoming deadlines (next 3 days), brief recap of what was ingested yesterday, items flagged for follow-up. Send the summary to Slack."
```

Or configure in `config.yaml` if Hermes supports declarative cron:

```yaml
cron:
  daily_summary:
    schedule: "0 7 * * *"
    timezone: Europe/London
    prompt: |
      Generate the daily PKM summary...
```

### 6.2 Weekly Summary (Monday 7:00 AM UK time)

```bash
hermes cron add \
  --schedule "0 7 * * 1" \
  --timezone "Europe/London" \
  --name "weekly-summary" \
  --prompt "Generate the weekly PKM summary. Include: week-in-review of ingested content and key themes, TODO completion rate and outstanding items, upcoming week deadlines, suggested items to review or archive. Send the summary to Slack."
```

### 6.3 Verify Cron Jobs

```bash
hermes cron list
```

---

## Phase 7: Backup Setup

### 7.1 Create Private Backup Repo

```bash
gh repo create hermes-backup --private --description "Hermes PKM agent backup"
```

### 7.2 Generate Fine-Grained PAT

1. Go to https://github.com/settings/tokens?type=beta
2. Click "Generate new token"
3. Name: `hermes-backup-vps`
4. Expiration: 90 days (set a reminder to rotate)
5. Repository access: "Only select repositories" > `hermes-backup`
6. Permissions: Contents (Read and write), Metadata (Read-only)
7. Generate and copy the token

### 7.3 Configure Git Auth for Backup

The backup process should use its own credentials, separate from the planning
repo PAT. Set up a git credential context for the backup repo:

```bash
# Option A: Use gh with a second auth for this specific repo
# (gh only supports one account per host, so this may need git credential store)

# Option B: Use git credential store for the backup repo specifically
git config --global credential.https://github.com/gilesknap/hermes-backup.helper store
echo "https://gilesknap:YOUR_PAT@github.com" >> ~/.git-credentials
chmod 600 ~/.git-credentials
```

**Note:** If you want both PATs to coexist (planning repo + backup repo), the
simplest approach is to use a single fine-grained PAT that has access to both
repos. Alternatively, use `gh` for the planning repo and git credential store
for the backup repo.

### 7.4 Initialize Backup Repo Locally

```bash
mkdir -p /opt/hermes-backup
cd /opt/hermes-backup
git init
git remote add origin https://github.com/gilesknap/hermes-backup.git
mkdir -p config data/db data/files
echo "# Hermes PKM Backup" > README.md
git add -A && git commit -m "Initial backup repo structure"
git push -u origin main
```

### 7.5 Create Backup Script

Write the backup script to `/opt/hermes-backup/backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

HERMES_HOME="$HOME/.hermes"
BACKUP_DIR="/opt/hermes-backup"
TIMESTAMP=$(date -u +%Y-%m-%dT%H:%MZ)

# 1. Safe SQLite backup (online, consistent snapshot)
DB_PATH="$HERMES_HOME/knowledge.db"   # adjust path based on actual Hermes DB location
if [ -f "$DB_PATH" ]; then
    sqlite3 "$DB_PATH" ".backup $BACKUP_DIR/data/db/knowledge.db"
fi

# 2. Copy config files (text, git-friendly)
rsync -a --delete \
    --exclude='*.db' \
    --exclude='*.db-wal' \
    --exclude='*.db-shm' \
    --exclude='sessions/' \
    --exclude='.env' \
    "$HERMES_HOME/" "$BACKUP_DIR/config/"

# 3. Copy ingested files (PDFs, images)
if [ -d "$HERMES_HOME/files" ]; then
    rsync -a --delete "$HERMES_HOME/files/" "$BACKUP_DIR/data/files/"
fi

# 4. Commit and push
cd "$BACKUP_DIR"
git add -A
if git diff --cached --quiet; then
    echo "[$TIMESTAMP] No changes to back up"
else
    git commit -m "backup $TIMESTAMP"
    git push
    echo "[$TIMESTAMP] Backup pushed"
fi
```

```bash
chmod +x /opt/hermes-backup/backup.sh
```

**Important:** The script excludes `.env` from backup (contains API keys).
Store tokens separately in a password manager.

### 7.6 Schedule Backup Cron (Every 6 Hours)

Use Hermes cron or system cron:

```bash
# System cron (more reliable -- runs even if Hermes is down)
crontab -e
```

Add:
```
0 */6 * * * /opt/hermes-backup/backup.sh >> /var/log/hermes-backup.log 2>&1
```

**Verify:**
```bash
# Run manually first
/opt/hermes-backup/backup.sh

# Check the remote repo
gh repo view gilesknap/hermes-backup --web
```

### 7.7 PAT Rotation Reminder

Add a TODO to the PKM agent itself:

> "Rotate GitHub backup PAT. Expires in 90 days. Go to github.com/settings/tokens,
> generate new fine-grained PAT for hermes-backup repo, update credentials on VPS."
> Recurrence: every 80 days. Priority: high.

---

## Phase 8: Systemd Service

Run Hermes as a persistent service so it survives reboots and restarts on crash.

### 8.1 Create Service File

```ini
# /etc/systemd/system/hermes-pkm.service
[Unit]
Description=Hermes PKM Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/hermes-agent
ExecStart=/opt/hermes-agent/venv/bin/hermes --gateway slack
Restart=on-failure
RestartSec=10
Environment=HERMES_HOME=/root/.hermes

[Install]
WantedBy=multi-user.target
```

**Note:** Running as root is expedient but not ideal. Consider creating a
dedicated `hermes` user for better security. The Hermes Docker container
uses UID 10000.

### 8.2 Enable and Start

```bash
systemctl daemon-reload
systemctl enable hermes-pkm
systemctl start hermes-pkm
```

### 8.3 Verify

```bash
systemctl status hermes-pkm
journalctl -u hermes-pkm -f   # follow logs
```

Send a test message to the bot in Slack. It should respond.

---

## Phase 9: Verification Checklist

Run through each of these to confirm the full system works:

### Ingestion Tests
- [ ] Send a URL in Slack DM -- agent should fetch, summarize, and confirm storage
- [ ] Upload a PDF -- agent should extract text, summarize, confirm
- [ ] Upload an image -- agent should describe it, confirm storage
- [ ] Send a plain text note -- agent should auto-categorize and store
- [ ] Send a code snippet -- agent should identify language, tag, store
- [ ] Create a TODO: "remind me to check backups next Tuesday, high priority"

### Retrieval Tests
- [ ] Search by topic: "find that article about X"
- [ ] Search by time: "what did I save yesterday?"
- [ ] Search by type: "show me my code snippets"
- [ ] Request original: "send me that PDF about Y" -- should get file attachment
- [ ] TODO query: "what's due this week?"

### Scheduled Summary Tests
- [ ] Trigger daily summary manually: ask agent to "generate today's summary"
- [ ] Verify cron: `hermes cron list` shows both daily and weekly jobs
- [ ] Wait for next scheduled run and confirm Slack delivery

### MCP Tests
- [ ] From Claude Code: use MCP tool to search PKM knowledge base
- [ ] From Claude Code: use MCP tool to ingest a note
- [ ] From claude.ai: same tests (if remote MCP configured)

### Backup Tests
- [ ] Run backup script manually: `/opt/hermes-backup/backup.sh`
- [ ] Check GitHub: repo should have new commit with DB snapshot
- [ ] Verify cron: `crontab -l` shows 6-hour schedule
- [ ] **Restore test:** on a test directory, clone backup repo and verify DB integrity:
  ```bash
  cd /tmp && git clone https://github.com/gilesknap/hermes-backup.git test-restore
  sqlite3 test-restore/data/db/knowledge.db "SELECT count(*) FROM items;"  # table name TBD
  ```

### Service Tests
- [ ] Reboot VPS: `systemctl reboot`
- [ ] After reboot: `systemctl status hermes-pkm` should show active
- [ ] Send a Slack message -- should respond within 30 seconds

---

## Recovery Procedure

If the VPS is lost, here's how to restore:

1. **Provision new VPS** (Ubuntu 24.04+, 2+ cores, 8GB+ RAM, 50GB+ disk)

2. **Run Phases 1-2** (prerequisites + Hermes install)

3. **Clone backup repo:**
   ```bash
   git clone https://github.com/gilesknap/hermes-backup.git /opt/hermes-backup
   ```

4. **Restore config:**
   ```bash
   mkdir -p ~/.hermes
   cp -r /opt/hermes-backup/config/* ~/.hermes/
   ```

5. **Restore data:**
   ```bash
   cp /opt/hermes-backup/data/db/knowledge.db ~/.hermes/knowledge.db
   cp -r /opt/hermes-backup/data/files/ ~/.hermes/files/
   ```

6. **Restore secrets:** Manually re-add `.env` from password manager:
   ```bash
   cat > ~/.hermes/.env << 'EOF'
   ANTHROPIC_API_KEY=sk-ant-...
   GOOGLE_GEMINI_API_KEY=AIza...
   SLACK_BOT_TOKEN=xoxb-...
   SLACK_APP_TOKEN=xapp-...
   EOF
   chmod 600 ~/.hermes/.env
   ```

7. **Run Phases 6-8** (cron, backup, systemd)

8. **Verify** with Phase 9 checklist

**Estimated recovery time:** 1-3 hours (mostly waiting for installs).
