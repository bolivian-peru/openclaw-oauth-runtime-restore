---
name: clawswap-v2
description: >
  Battle-tested migration guide: OpenClaw → Claude Code native runtime.
  Based on a real production migration of a Telegram support bot with
  50+ MCP tools, session memory, and customer-facing AI. Covers the
  approaches that failed, the approach that worked, and every pitfall
  encountered in production. Swaps API-credit billing for Max-plan
  OAuth. Non-destructive — all skills, state, configs stay untouched.
trigger: |
  Use when the user says: /clawswap, migrate openclaw, swap runtime,
  move off API credits, convert to claude code, clawback, or wants to
  convert any OpenClaw bot to use claude -p with OAuth subscription.
allowed-tools:
  - "Bash(*)"
  - "Read(*)"
  - "Write(*)"
  - "Edit(*)"
requires:
  bins: [bash, curl, node, npm, jq]
  runtime:
    - "Anthropic Claude Max plan account (for OAuth token)"
    - "Existing OpenClaw deployment"
    - "Node.js >= 20"
---

# ClawSwap v2 — OpenClaw → Claude Code Runtime Migration

## Battle-tested. Production-proven. Real receipts.

This is the comprehensive, long-form migration guide based on an actual production
conversion of a Telegram support bot from OpenClaw to Claude Code native runtime.
The bot had 50+ tools (API endpoints + workspace file tools), multi-conversation
session memory, customer-facing AI support with CRM integration, and was serving
real users at the time of migration.

The migration took approximately 20–40 minutes of active work once the correct
approach was identified. This document covers the approaches that failed first,
so you don't waste time repeating them.

Built on [os-moda](https://github.com/bolivian-peru/os-moda/). Works on any
OpenClaw deployment.

---

## What Changed vs What Stayed

This is the most important table. Read it before anything else.

| layer | before (OpenClaw) | after (Claude Code) | status |
|---|---|---|---|
| billing | API credits per token ($20–60/day) | Max plan quota (fixed monthly, $0 extra) | CHANGED |
| auth | `ANTHROPIC_API_KEY` env var | `~/.claude/.credentials.json` OAuth file | CHANGED |
| invocation | `openclaw run <skill>` / agentd IPC | `claude -p "$PROMPT"` subprocess | CHANGED |
| system prompt | OpenClaw system prompt config | `CLAUDE.md` in workspace dir (auto-loaded) | CHANGED |
| tools | OpenClaw native tool system | MCP tools via `.mcp.json` or `--mcp-config` | CHANGED |
| gateway | openclaw-gateway / grammy bot | bash bridge polling `getUpdates` → `claude -p` | CHANGED |
| process manager | PM2 / systemd (OpenClaw managed) | PM2 / systemd (you manage directly) | CHANGED |
| **skill scripts** | `~/.openclaw/skills/*` | **UNTOUCHED** | same |
| **skill configs** | `~/.openclaw/<skill>/config/*` | **UNTOUCHED** | same |
| **skill state** | `~/.openclaw/<skill>/state/*` | **UNTOUCHED** | same |
| **secrets** | secrets directory | **UNTOUCHED** | same |
| **database** | MongoDB / whatever you use | **UNTOUCHED** | same |
| **telegram bot token** | same token | **UNTOUCHED** | same |
| **external integrations** | RPC, webhooks, APIs | **UNTOUCHED** | same |

The migration is aggressive on the **runtime layer** (no more API credit billing,
no more agentd gateway) and completely non-destructive on the **skill layer**
(everything in `~/.openclaw/skills/` keeps working as-is).

---

## Architecture After Migration

```
                    ┌──────────────────────────────┐
                    │  /root/.claude/               │
                    │   .credentials.json (OAuth)   │
                    │   settings.json (perms)       │
                    │   projects/-root-workspace/   │  ← session jsonl per conv
                    └──────────────┬───────────────┘
                                   │
   /root/workspace/  (always cwd)  │
   ├─ CLAUDE.md       (auto-loaded system prompt)
   ├─ .claude/        (per-project settings)
   ├─ .mcp.json       (MCP tool servers)
   └─ .openclaw → /root/.openclaw (symlink to legacy skills)
                                   │
               ┌───────────────────┴────────────────┐
               │                                    │
    /opt/claude-code/                      /root/.openclaw/skills/
    ├─ telegram-bridge.sh ──┐              ├─ skill-a/
    ├─ heartbeat.sh      ───┤              ├─ skill-b/
    ├─ review.sh         ───┤ each wrapper └─ skill-c/
    └─ node_modules/        │ builds a prompt
              ▼             │ and pipes into
    @anthropic-ai/claude-code    claude -p "$PROMPT"
                                   │
                                   ▼
                        Anthropic OAuth (Max plan quota)
                                   │
                                   ▼
                        PM2 or systemd (your choice)
```

---

## Rules

- NEVER delete anything under `~/.openclaw/` — the migration is non-destructive
- NEVER set `ANTHROPIC_API_KEY` anywhere in the environment — if both OAuth and API key exist, the API key wins and you fall back to credit billing. This is the single most common mistake.
- NEVER use `ANTHROPIC_AUTH_TOKEN` env var — it does NOT work for OAuth tokens (see "What We Tried That Failed" section below)
- Always confirm with the user before writing systemd units, NixOS config, or PM2 ecosystem files
- Ask which skills exist and how they're currently invoked before generating any wrappers
- Test with `claude -p "say only: pong"` before doing anything else — if that doesn't return "pong", stop and fix auth first

---

## Step -1 — Preserve Your Existing Workspace (DO THIS FIRST)

Before changing anything, create a full snapshot of your current OpenClaw environment.
The migration is non-destructive by design — your skills, state, and configs all stay
in place — but having a snapshot means you can diff against it after migration to
verify that absolutely nothing was lost or modified.

```bash
# 1. Snapshot the entire OpenClaw directory tree
tar czf /root/openclaw-backup-$(date +%F).tar.gz \
    ~/.openclaw/ \
    /var/lib/secrets/ \
    2>/dev/null

# 2. Capture every running process, service, and cron entry
systemctl list-units --type=service --state=running --no-pager > /root/pre-migration-services.txt
pm2 list 2>/dev/null > /root/pre-migration-pm2.txt
crontab -l 2>/dev/null > /root/pre-migration-crontab.txt

# 3. Snapshot your bot's system prompt / personality config
# (wherever OpenClaw stores it — skill files, SOUL.md, playbook, etc.)
find ~/.openclaw -name '*.md' -o -name 'system-prompt*' -o -name 'SOUL*' -o -name 'PLAYBOOK*' \
    2>/dev/null | xargs tar czf /root/openclaw-prompts-backup.tar.gz 2>/dev/null

# 4. Record the Telegram bot's current offset and webhook state
BOT_TOKEN="<your-bot-token>"
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/getWebhookInfo" > /root/pre-migration-webhook.json
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/getUpdates?offset=-1&limit=1" > /root/pre-migration-offset.json

# 5. If your bot has a database, snapshot the connection info
echo "DB connection:" > /root/pre-migration-db-info.txt
grep -r 'MONGO\|DATABASE\|DB_URL' ~/.openclaw/ /etc/environment 2>/dev/null >> /root/pre-migration-db-info.txt

# 6. Record all environment variables the current bot depends on
env | grep -Ei 'TOKEN|KEY|SECRET|MONGO|DB|API|BOT|TELEGRAM|WEBHOOK' \
    > /root/pre-migration-env.txt
chmod 600 /root/pre-migration-env.txt

# 7. List all tools the bot currently has access to
# (this is what you'll need to recreate as .mcp.json later)
ls -la ~/.openclaw/tools/ ~/.openclaw/mcp/ 2>/dev/null
find ~/.openclaw -name 'tools.json' -o -name 'mcp*.json' 2>/dev/null | \
    xargs cat 2>/dev/null > /root/pre-migration-tools.txt
```

**What you're preserving and why**:

- **Skill scripts** (`~/.openclaw/skills/*`) — your domain logic. NOT touched during migration.
- **State files** (`~/.openclaw/<skill>/state/*`) — accumulated runtime data, logs, sessions. NOT touched.
- **Config files** (`~/.openclaw/<skill>/config/*`) — tuned parameters, thresholds, settings. NOT touched.
- **Secrets** — API keys, wallet keys, bot tokens, DB credentials. NOT touched. The only credential that changes is the Anthropic auth (API key → OAuth).
- **System prompt / personality** — your bot's personality, playbook, rules. These get MIGRATED into `CLAUDE.md` format, but the content stays the same. The backup lets you verify nothing was lost in translation.
- **Tool definitions** — your tools / API endpoints. These get MIGRATED into `.mcp.json` format. The backup lets you verify all tools are accounted for.
- **Telegram state** — offset and webhook info so the new bridge picks up exactly where the old gateway left off.

**After migration, verify with**:

```bash
# Compare skill files — should be identical
diff <(tar tf /root/openclaw-backup-*.tar.gz | sort) \
     <(find ~/.openclaw -type f | sed 's|^/root/||' | sort)

# Verify no state files were modified after the backup
find ~/.openclaw -name '*.json' -newer /root/openclaw-backup-*.tar.gz 2>/dev/null
# Should return empty
```

The goal is **exact functional parity**: same bot personality, same tools, same data,
same Telegram behavior, same everything — just a different binary processing the prompts
and a different line on the bill.

---

## What We Tried That Failed

These are the approaches we attempted before finding the one that works. Documenting
them here so nobody wastes time repeating our mistakes.

### Failed Approach 1: Anthropic SDK with ANTHROPIC_AUTH_TOKEN

We first tried building the bot using the Anthropic SDK directly (`@anthropic-ai/sdk`)
with the OAuth access token passed as `ANTHROPIC_AUTH_TOKEN` env var. This seemed
logical — the token starts with `sk-ant-oat01-`, which looks like an API credential.

**What happened**: Claude Code sent the token as a `Bearer` header to `api.anthropic.com`,
which rejected it. OAuth access tokens are NOT the same as API keys. The Anthropic API
expects either a proper `x-api-key` header (for API keys) or the OAuth flow handled
internally by the Claude Code CLI. There is no way to use an OAuth access token directly
with the SDK.

**The error**: `API Error: Unable to connect to API (ConnectionRefused)` or authentication
failures depending on which endpoint was hit.

**Lesson**: OAuth tokens only work through the Claude Code CLI binary (`claude` / `claude -p`).
They do NOT work with the Anthropic SDK, REST API calls, or any env var approach. The CLI
handles the full OAuth flow internally — token storage, refresh, scoping.

### Failed Approach 2: claude auth login on headless server

We tried running `claude auth login --claudeai` on the headless VPS. It generated a URL
to visit in a browser. The user copied the URL, but the authentication failed with
"Invalid code_challenge_method" — likely due to URL truncation during copy-paste from
the SSH terminal.

**Lesson**: Interactive OAuth login from a headless server is fragile. URLs are long and
easily mangled in terminal copy-paste. Use Option C (programmatic credential file) instead.

### What Actually Works: Writing credentials.json directly

If you have an OAuth access token (the `sk-ant-oat01-...` string), you can write the
credentials file directly on the server. No browser. No interactive login. No SCP from
another machine. Just write the JSON file.

This is documented as Step 3 Option C below.

---

## Step 0 — Inventory the Source Box

Before touching anything, capture what the existing OpenClaw setup is doing so nothing
is lost in the swap. This step is critical — if you skip it, you'll miss invocation
points and things will silently stop running.

```bash
# 1. List every openclaw / agentd invocation point
systemctl list-unit-files --no-pager | grep -Ei 'openclaw|agentd'
pm2 list 2>/dev/null | grep -Ei 'openclaw|agentd|bot'
crontab -l 2>/dev/null
ls -la ~/.openclaw/skills/

# 2. Capture the working state (pause-safe, no stop yet)
ls ~/.openclaw/*/state/ ~/.openclaw/*/config/ 2>/dev/null
find ~/.openclaw -name 'settings.json' -maxdepth 4

# 3. Note any secrets the skills need
find ~/.openclaw -name '*.key' -o -name 'secrets*' -o -name '.env*' 2>/dev/null
ls /var/lib/secrets/ 2>/dev/null  # or wherever your secrets live

# 4. Check what Telegram bots are running and their tokens
ps aux | grep -i 'telegram\|grammy\|openclaw' | grep -v grep

# 5. Check for any webhooks set on the Telegram bot
# (webhooks intercept getUpdates polling — you MUST know about them)
BOT_TOKEN="<your-bot-token>"
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/getWebhookInfo" | jq .

# 6. Note the current getUpdates offset (if using polling)
# The old OpenClaw gateway maintains an offset. If your new bridge starts
# with a stale offset, it will miss messages.
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/getUpdates?offset=-1&limit=1" | jq .
```

Write the output to a scratch file. You will reference it when rewriting wrappers and
configuring the Telegram bridge.

**IMPORTANT**: Note the Telegram `update_id` offset. When the old gateway stops and your
new bridge starts, it needs to continue from the correct offset or start fresh. Starting
with a stale high offset means the bridge will poll forever and never see new messages.
Starting with offset 0 or -1 gets the latest pending update.

---

## Step 1 — Install Claude Code CLI

Claude Code is shipped as an npm package (`@anthropic-ai/claude-code`). Install it under
`/opt/claude-code/` rather than globally so you can pin a specific version and process
managers can reference a fixed path.

```bash
# Verify Node.js version (must be >= 20)
node --version

# Create the install directory
sudo mkdir -p /opt/claude-code
cd /opt/claude-code

# Pin the version in package.json for deterministic installs
sudo tee package.json >/dev/null <<'JSON'
{
  "name": "claude-code",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "@anthropic-ai/claude-code": "^2.1.92"
  }
}
JSON

sudo npm install

# Symlink to a common PATH location
sudo mkdir -p /root/.local/bin
sudo ln -sf /opt/claude-code/node_modules/.bin/claude /root/.local/bin/claude

# Smoke test — just the version, no auth needed
/root/.local/bin/claude --version
# Expected output: 2.1.92 (Claude Code) or similar
```

If you're on NixOS, Node.js comes from nixpkgs. On Debian/Ubuntu, use the nodesource
repository for Node 20+.

---

## Step 2 — OAuth Authentication

This is the most critical step and the one with the most pitfalls. There are three
options. Option C is what we used in production and is the most reliable for headless
servers.

### Option A — Interactive Login (machine has a browser)

```bash
HOME=/root /root/.local/bin/claude
# Inside the REPL, type:
/login
# Follow the URL printed to your terminal
# Complete the OAuth flow in your browser
# Paste the returned code back into the REPL
# The CLI writes ~/.claude/.credentials.json automatically
/exit
```

This is the simplest method but requires a browser-capable terminal.

### Option B — SCP from a workstation

1. Run `claude /login` on a workstation where you can open a browser
2. Authenticate with the **same Anthropic account that owns the Max plan**
3. Verify the credentials:

```bash
cat ~/.claude/.credentials.json | jq .claudeAiOauth.subscriptionType
# Should output: "max"
```

4. Copy to the target server:

```bash
scp ~/.claude/.credentials.json vps:/root/.claude/.credentials.json
ssh vps "chmod 600 /root/.claude/.credentials.json && chown root:root /root/.claude/.credentials.json"
```

### Option C — Programmatic Credential File (recommended for headless servers)

This is what we actually used. If you have an OAuth access token (`sk-ant-oat01-...`),
you can write the credentials file directly without any browser interaction.

Where do you get the access token? From an existing OpenClaw installation, from a
previous `claude /login` session on any machine, or from the Anthropic OAuth flow
if you've implemented it elsewhere.

```bash
# Create the credentials directory
mkdir -p /root/.claude

# Write the credentials file directly
cat > /root/.claude/.credentials.json << 'EOF'
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-oat01-YOUR_ACCESS_TOKEN_HERE",
    "refreshToken": "sk-ant-ort01-YOUR_REFRESH_TOKEN_HERE",
    "expiresAt": 1775774290080,
    "scopes": [
      "user:inference",
      "user:profile",
      "user:sessions:claude_code"
    ],
    "subscriptionType": "max",
    "rateLimitTier": "default_claude_max_20x"
  }
}
EOF

# Lock down permissions
chmod 600 /root/.claude/.credentials.json
chown root:root /root/.claude/.credentials.json
```

**Critical fields explained**:

- `accessToken` — The `sk-ant-oat01-...` string. This is what authenticates API calls.
  If you only have the access token and no refresh token, this still works — but you'll
  need to regenerate when the access token expires. The CLI normally handles refresh
  automatically using the refreshToken.

- `refreshToken` — The `sk-ant-ort01-...` string. Used by the CLI to automatically
  refresh the access token before expiry. If you don't have it, set it to an empty
  string — the access token will work until it expires.

- `expiresAt` — Unix timestamp in milliseconds for when the access token expires.
  Set it to a far-future value if you're not sure. The CLI checks this before each call.

- `subscriptionType` — Must be `"max"` for Max plan. This tells the CLI which rate
  limit tier to expect.

- `rateLimitTier` — Either `"default_claude_max_20x"` (Max 20x plan) or
  `"default_claude_max_5x"` (Max 5x plan). Determines your token budget.

### Verify Authentication

This is the single most important test. If this doesn't work, NOTHING else will work.
Run it before proceeding to any other step.

```bash
# Verify the credentials file is readable
jq .claudeAiOauth.subscriptionType /root/.claude/.credentials.json
# Expected: "max"

# Verify claude -p actually works with these credentials
HOME=/root /root/.local/bin/claude -p "say only the word: pong" 2>&1 | head
# Expected: pong
```

If you see "pong", authentication is working and every subsequent `claude -p` call
will bill against your Max plan subscription instead of API credits.

If you see "please run /login" or any error, the credentials file is missing, malformed,
or the token has expired. Fix this before doing anything else.

**THE CRITICAL WARNING**: Do NOT also set `ANTHROPIC_API_KEY` anywhere in the environment.
If both are present, the API key takes priority and you silently fall back to per-token
credit billing. This is the most common mistake. Audit your environment:

```bash
# Check for stray API key exports
grep -r 'ANTHROPIC_API_KEY' /etc/ /root/ ~/.bashrc ~/.zshrc ~/.profile 2>/dev/null
env | grep ANTHROPIC
```

If you find any, remove them.

---

## Step 3 — Workspace + CLAUDE.md

Claude Code looks for `CLAUDE.md` in the current working directory at launch and
prepends its contents to the system prompt. This is how the runtime "knows" about
the host's services, paths, rules, and personality without being told every invocation.

This replaces OpenClaw's system prompt mechanism entirely. Every time `claude -p` is
called from the workspace directory, it reads `CLAUDE.md` fresh. No restart needed —
edit the file and the next invocation picks it up immediately.

```bash
# Create the workspace
sudo mkdir -p /root/workspace
cd /root/workspace

# Symlink the legacy openclaw tree (so skill scripts remain reachable)
sudo ln -sf /root/.openclaw .openclaw 2>/dev/null
```

### Writing CLAUDE.md

The CLAUDE.md should contain everything the AI needs to know about its role, the host,
the services it manages, and the rules it must follow. This is your bot's brain.

For a Telegram support bot, the CLAUDE.md should include:

1. **Identity section** — who the bot is, its name, its role
2. **Communication style rules** — this is where you control the bot's personality
3. **Context gathering rules** — what data to fetch BEFORE responding to any user
4. **Available tools** — what MCP tools exist and when to use them
5. **Playbook** — common scenarios and how to handle them
6. **Safety rules** — what the bot must never do

**Real-world insight on bot personality**: In our production migration, the bot initially
gave generic, corporate-sounding AI responses. We had to explicitly add rules like:

```markdown
## Communication Style

- mirror the customer's writing style. if they write lowercase and casual, respond the same way. if they're formal and professional, match that.
- default tone: lowercase, no fluff, straight to the point
- NEVER use these phrases: "I apologize for the inconvenience", "Thank you for your patience", "I understand your frustration", "Let me look into that for you"
- NEVER use exclamation marks in customer responses
- fix first, report results second. don't say "let me check" — just check and report what you found
- short responses. 1-3 sentences unless the customer needs a detailed explanation
- if you make a small typo or imperfection, leave it — it reads more human

## Before Responding to ANY Message

MANDATORY: Before typing a single word of response, gather full context:
1. Look up the user's account, purchase history, and all open tickets
2. Read the CRM notes for any prior interactions
3. Check the current status of whatever they're asking about
4. THEN respond with full awareness — never look stupid by missing context that was one tool call away
```

This was the single biggest improvement to the bot's quality. Without it, `claude -p`
gives helpful but generic AI responses. With it, the bot reads like a human support agent
who knows the customer's full history.

---

## Step 4 — Permissions

Two settings files are honored: a global one at `~/.claude/settings.json` and an
optional per-project one at `<cwd>/.claude/settings.json`. For an unattended bot or
server deployment, you want every tool unconditionally allowed.

```bash
# Global settings
sudo mkdir -p /root/.claude
sudo tee /root/.claude/settings.json >/dev/null <<'JSON'
{
  "permissions": {
    "allow": [
      "Bash(*)",
      "Read(*)",
      "Write(*)",
      "Edit(*)",
      "Glob(*)",
      "Grep(*)",
      "Agent(*)"
    ],
    "deny": []
  },
  "enableAllProjectMcpServers": true
}
JSON

# Per-project settings (lives in the workspace so rules travel with cwd)
sudo mkdir -p /root/workspace/.claude
sudo tee /root/workspace/.claude/settings.json >/dev/null <<'JSON'
{
  "permissions": {
    "allow": [
      "Bash(*)", "Read(*)", "Write(*)", "Edit(*)",
      "Glob(*)", "Grep(*)", "Agent(*)"
    ],
    "deny": []
  }
}
JSON
```

Without these settings files, `claude -p` will prompt for permission on every Bash
command, Read operation, etc. On a headless server with no TTY, this causes the process
to hang forever waiting for input that will never come.

---

## Step 5 — MCP Tools Configuration

If your OpenClaw bot had tools (API endpoints, database queries, file operations, etc.),
you need to expose them as MCP (Model Context Protocol) tool servers. This replaces
OpenClaw's native tool system.

MCP config lives in `/root/workspace/.mcp.json`. The Claude Code CLI reads this file
on startup and connects to all configured MCP servers.

Example `.mcp.json` for a bot with an HTTP API tool server:

```json
{
  "mcpServers": {
    "my-api": {
      "command": "node",
      "args": ["/home/projects/my-bot/src/mcp-server.js"],
      "env": {
        "API_BASE_URL": "http://localhost:5000",
        "MONGO_URI": "mongodb://..."
      }
    }
  }
}
```

If your tools are simple HTTP endpoints, you can also use `claude -p` with `--mcp-config`
flag to point to the config file, or rely on the `.mcp.json` in the workspace directory.

**Real-world insight**: In our production migration, we had 50+ tools (API endpoints for
user management, ticket handling, balance operations, etc., plus workspace file
read/write tools). All of these were exposed through a single MCP server that proxied
to the existing REST API. The API code itself was completely untouched.

---

## Step 6 — Telegram Bridge

This is the heart of a Telegram-based bot migration. OpenClaw handled Telegram polling
through its gateway (often using grammy or node-telegram-bot-api). In the Claude Code
approach, you replace this with a bash daemon that:

1. Polls Telegram `getUpdates` in a loop
2. Routes free-text messages to `claude -p --resume <session-id>`
3. Handles slash commands directly (no LLM needed for `/menu`, `/help`, etc.)
4. Persists session IDs per conversation for multi-turn memory
5. Sends responses back via Telegram `sendMessage`

### The Bridge Script

Create `/opt/claude-code/telegram-bridge.sh`:

```bash
#!/usr/bin/env bash
#
# Telegram Bridge — Routes messages to claude -p with session resumption
#
export PATH=/root/.local/bin:/usr/local/bin:/usr/bin:/bin:$PATH
export HOME=/root

BOT_TOKEN="<your-bot-token>"
CHAT_IDS="<comma-separated-allowed-chat-ids>"  # security: only respond to these chats
LOG_DIR="/root/.claude-bot/logs"
CONV_DIR="/root/.claude/channels/telegram/conversations"
WORKSPACE="/root/workspace"

mkdir -p "$LOG_DIR" "$CONV_DIR"

log() { echo "[$(date '+%F %T')] $*" >> "$LOG_DIR/telegram.log"; }
log "=== Telegram Bridge starting ==="

OFFSET=0  # start fresh — see note below about offsets

send_tg() {
    local chat_id="$1" text="$2"
    # Telegram messages have a 4096 char limit
    if [[ ${#text} -gt 4000 ]]; then
        text="${text:0:4000}... (truncated)"
    fi
    curl -s "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        -d "chat_id=$chat_id" \
        --data-urlencode "text=$text" >/dev/null 2>&1
}

get_session_id() {
    local name="${1:-default}"
    cat "$CONV_DIR/$name/session-id.txt" 2>/dev/null
}

save_session_id() {
    local name="${1:-default}" sid="$2"
    mkdir -p "$CONV_DIR/$name"
    echo "$sid" > "$CONV_DIR/$name/session-id.txt"
}

CYCLE=0

while true; do
    # Poll for updates (30-second long poll)
    UPDATES=$(curl -s -m 40 \
        "https://api.telegram.org/bot${BOT_TOKEN}/getUpdates?offset=${OFFSET}&timeout=30&allowed_updates=%5B%22message%22%5D")

    # Parse each update
    echo "$UPDATES" | jq -c '.result[]?' 2>/dev/null | while read -r update; do
        UPDATE_ID=$(echo "$update" | jq -r '.update_id')
        CHAT_ID=$(echo "$update" | jq -r '.message.chat.id')
        TEXT=$(echo "$update" | jq -r '.message.text // empty')
        FROM=$(echo "$update" | jq -r '.message.from.first_name // "user"')

        # Advance offset past this update
        OFFSET=$((UPDATE_ID + 1))

        # Skip empty messages
        [[ -z "$TEXT" ]] && continue

        # Security: only respond to allowed chats
        if [[ ! ",$CHAT_IDS," == *",$CHAT_ID,"* ]]; then
            log "Ignored message from unauthorized chat $CHAT_ID"
            continue
        fi

        log "Message from $FROM ($CHAT_ID): $TEXT"

        # Handle slash commands (no LLM needed)
        case "$TEXT" in
            /menu|/help|/start)
                send_tg "$CHAT_ID" "Commands:
/new [name] - start fresh conversation
/list - show conversations
/switch <name> - switch conversation
/current - show active conversation
/status - system status"
                continue
                ;;
            /new*)
                NAME=$(echo "$TEXT" | awk '{print $2}')
                [[ -z "$NAME" ]] && NAME=$(date +%b%d-%H%M | tr '[:upper:]' '[:lower:]')
                mkdir -p "$CONV_DIR/$NAME"
                echo "$NAME" > "$CONV_DIR/active.txt"
                send_tg "$CHAT_ID" "Started conversation: $NAME"
                log "New conversation: $NAME"
                continue
                ;;
            /list)
                LIST=$(ls "$CONV_DIR" 2>/dev/null | grep -v active.txt | tr '\n' ', ')
                ACTIVE=$(cat "$CONV_DIR/active.txt" 2>/dev/null || echo "default")
                send_tg "$CHAT_ID" "Conversations: $LIST
Active: $ACTIVE"
                continue
                ;;
            /switch*)
                NAME=$(echo "$TEXT" | awk '{print $2}')
                if [[ -d "$CONV_DIR/$NAME" ]]; then
                    echo "$NAME" > "$CONV_DIR/active.txt"
                    send_tg "$CHAT_ID" "Switched to: $NAME"
                else
                    send_tg "$CHAT_ID" "Conversation not found: $NAME"
                fi
                continue
                ;;
            /current)
                ACTIVE=$(cat "$CONV_DIR/active.txt" 2>/dev/null || echo "default")
                SID=$(get_session_id "$ACTIVE")
                send_tg "$CHAT_ID" "Active: $ACTIVE
Session: ${SID:-none}"
                continue
                ;;
        esac

        # Route free text to claude -p
        ACTIVE=$(cat "$CONV_DIR/active.txt" 2>/dev/null || echo "default")
        SESSION_ID=$(get_session_id "$ACTIVE")

        # Send "typing" indicator
        curl -s "https://api.telegram.org/bot${BOT_TOKEN}/sendChatAction" \
            -d "chat_id=$CHAT_ID" -d "action=typing" >/dev/null 2>&1

        # Call claude -p with or without session resume
        if [[ -n "$SESSION_ID" ]]; then
            RESPONSE=$(cd "$WORKSPACE" && claude -p "$TEXT" --resume "$SESSION_ID" 2>/dev/null)
        else
            RESPONSE=$(cd "$WORKSPACE" && claude -p "$TEXT" 2>/dev/null)
        fi

        # Capture the latest session ID for next message
        LATEST=$(ls -t /root/.claude/sessions/ 2>/dev/null | head -1)
        [[ -n "$LATEST" ]] && save_session_id "$ACTIVE" "$LATEST"

        # Send response (or error)
        if [[ -n "$RESPONSE" ]]; then
            send_tg "$CHAT_ID" "$RESPONSE"
            log "Response sent (${#RESPONSE} chars)"
        else
            send_tg "$CHAT_ID" "no response generated. try rephrasing."
            log "Empty response for: $TEXT"
        fi
    done

    # Periodic cleanup (every ~30 polls)
    CYCLE=$((CYCLE + 1))
    if (( CYCLE % 30 == 0 )); then
        # Kill stale claude -p processes (older than 5 min)
        find /proc -maxdepth 1 -name '[0-9]*' -mmin +5 2>/dev/null | while read -r p; do
            cmdline=$(cat "$p/cmdline" 2>/dev/null | tr '\0' ' ')
            if [[ "$cmdline" == *"claude"*"-p"* ]]; then
                kill "$(basename "$p")" 2>/dev/null
            fi
        done
        # Prune old sessions (older than 7 days)
        find /root/.claude/sessions -mtime +7 -delete 2>/dev/null
        log "Heartbeat: polls=$CYCLE, offset=$OFFSET"
    fi
done
```

Make it executable:
```bash
sudo chmod +x /opt/claude-code/telegram-bridge.sh
```

### Critical Note: Telegram Offset

When switching from OpenClaw's gateway to this bridge, you MUST handle the poll offset
correctly. The old gateway maintained its own offset. If you start the new bridge with
the old offset, it will skip all messages sent during the transition.

**Safest approach**: Start with `OFFSET=0` in the script. Telegram will re-deliver the
most recent unacknowledged update. The bridge will process it and advance the offset.

**If messages are still being eaten**: Check that the old OpenClaw gateway is fully stopped.
Two processes polling the same bot token will race for updates — whichever acknowledges
first wins, and the other never sees the message.

```bash
# Verify no competing processes
ps aux | grep -i 'openclaw\|grammy\|telegram' | grep -v grep

# Verify no webhook is intercepting (webhooks take priority over polling)
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/getWebhookInfo" | jq .result.url
# Should be empty string ""

# If a webhook is set, delete it:
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/deleteWebhook"
```

---

## Step 7 — Process Management

You have two choices: PM2 (what we used in production) or systemd. Both work fine.
PM2 is more convenient for rapid iteration; systemd is more robust for permanent deployment.

### Option A — PM2

```bash
# Start the bridge
pm2 start /opt/claude-code/telegram-bridge.sh --name claude-telegram

# Save the PM2 process list so it survives reboot
pm2 save
pm2 startup  # generates the systemd unit for PM2 itself

# Monitor
pm2 logs claude-telegram --lines 50
pm2 monit
```

### Option B — Systemd

```ini
# /etc/systemd/system/claude-telegram.service
[Unit]
Description=Claude Code Telegram Bridge
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/opt/claude-code/telegram-bridge.sh
Restart=always
RestartSec=10
Environment=PATH=/root/.local/bin:/usr/local/bin:/usr/bin:/bin
Environment=HOME=/root

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now claude-telegram.service
```

### Disable the Old Gateway

This is mandatory. Two processes polling the same Telegram bot will steal each other's
messages.

```bash
# If it was PM2:
pm2 stop openclaw-gateway 2>/dev/null && pm2 delete openclaw-gateway 2>/dev/null

# If it was systemd:
sudo systemctl disable --now openclaw-gateway 2>/dev/null
sudo systemctl disable --now openclaw-gateway 2>/dev/null
```

---

## Step 8 — Skill Wrapper Conversion

For each `openclaw <skill>` invocation in your inventory, create a shell wrapper in
`/opt/claude-code/`. Three patterns cover ~95% of cases:

### Pattern A — Health Check (no LLM unless alert)

For monitoring scripts that only need Claude when something is wrong. Cheap — uses
zero LLM quota in steady state.

```bash
#!/usr/bin/env bash
export PATH=/root/.local/bin:$PATH HOME=/root
LOG="/root/.openclaw/<skill>/state/heartbeat.log"

cpu=$(awk '{print $1}' /proc/loadavg)
mem=$(free | awk '/Mem:/{printf "%.0f", $3/$2*100}')
disk=$(df / | awk 'NR==2{print $5}' | tr -d %)
down=""
for svc in <critical-service-1> <critical-service-2>; do
    systemctl is-active "$svc" >/dev/null 2>&1 || down="$down $svc"
done
echo "$(date -u +%FT%TZ) load=$cpu mem=${mem}% disk=${disk}%${down:+ DOWN:$down}" >> "$LOG"

# Only invoke Claude when something is actually wrong
if (( mem > 85 )) || (( disk > 90 )) || [[ -n "$down" ]]; then
    claude -p "ALERT: load=$cpu mem=${mem}% disk=${disk}% down=$down. Investigate and report." \
        --max-turns 3 >> "$LOG" 2>&1
fi
```

### Pattern B — Periodic LLM Review with Telegram Output

For jobs that run on a schedule, read state, ask Claude to analyze, and forward a
summary to Telegram.

```bash
#!/usr/bin/env bash
export PATH=/root/.local/bin:$PATH HOME=/root
STATE_DIR="/root/.openclaw/<skill>/state"
BOT_TOKEN=$(cat /var/lib/secrets/telegram-bot-token 2>/dev/null)
CHAT_ID="<your-chat-id>"

send_tg() {
    [[ -z "$BOT_TOKEN" ]] && return
    curl -s "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        -d "chat_id=$CHAT_ID" --data-urlencode "text=$1" >/dev/null 2>&1
}

# Gather context from state files
DATA=$(cat "$STATE_DIR"/*.json 2>/dev/null)
RECENT=$(tail -25 "$STATE_DIR/run.log" 2>/dev/null)

PROMPT="You are the automated reviewer for <skill>.

CURRENT STATE:
$DATA

RECENT LOG:
$RECENT

TASKS:
1. Analyze current state.
2. If you see issues, explain them.
3. Emit a Telegram summary block:
---TELEGRAM---
3-5 line summary
---END---"

cd /root/workspace
RESPONSE=$(claude -p "$PROMPT" 2>/dev/null)
echo "$(date -u +%FT%TZ) review" >> "$STATE_DIR/review.log"
echo "$RESPONSE" >> "$STATE_DIR/review.log"

# Extract and send Telegram block
TG_MSG=$(echo "$RESPONSE" | sed -n '/---TELEGRAM---/,/---END---/p' | grep -v '^---')
[[ -n "$TG_MSG" ]] && send_tg "$TG_MSG"
```

### Pattern C — Pure Worker (no LLM)

For background jobs that don't need Claude at all — just a node process with the
right environment.

```bash
#!/usr/bin/env bash
export PATH=/root/.local/bin:$PATH HOME=/root
cd /root/.openclaw/skills/<skill>
exec node scripts/<worker>.mjs
```

---

## Step 9 — Verification Checklist

Run ALL of these after deployment. If any fail, fix before considering the migration done.

```bash
# 1. CLI is installed and authenticated
/root/.local/bin/claude --version
jq .claudeAiOauth.subscriptionType /root/.claude/.credentials.json
# Expected: version number ; "max"

# 2. Smoke test — must NOT prompt for browser or permission
HOME=/root /root/.local/bin/claude -p "say only the word: pong" 2>&1 | head
# Expected: pong

# 3. Auto-context is loading (CLAUDE.md is being read)
HOME=/root cd /root/workspace && /root/.local/bin/claude -p "what is your name and role? one line" 2>&1
# Expected: references info from CLAUDE.md

# 4. Tool permissions work (no gate)
HOME=/root cd /root/workspace && /root/.local/bin/claude -p "run: echo hello && date -u" 2>&1
# Expected: hello + timestamp, no permission prompt

# 5. Process manager shows the bridge running
pm2 list 2>/dev/null | grep telegram
# OR
systemctl status claude-telegram --no-pager | head -5

# 6. Bridge logs show polling activity
tail -20 /root/.claude-bot/logs/telegram.log
# Expected: "Starting poll loop" + heartbeat entries

# 7. Telegram round-trip — send /menu from your Telegram client
# Expected: the menu reply appears in chat

# 8. Session persistence — send a free-text message, then another
ls -t /root/.claude/sessions/ 2>/dev/null | head
# Expected: freshly-created session files

# 9. Old gateway is dead
pm2 list 2>/dev/null | grep openclaw  # should show nothing or "stopped"
systemctl status openclaw-gateway --no-pager 2>&1 | head -3  # should be inactive

# 10. No ANTHROPIC_API_KEY in the environment
env | grep ANTHROPIC_API_KEY
grep -r 'ANTHROPIC_API_KEY' /etc/ /root/.bashrc /root/.zshrc /root/.profile 2>/dev/null
# Expected: empty — no matches
```

---

## Step 10 — Bot Personality Tuning (post-deployment)

This step happens AFTER the bot is live and responding. You WILL need to tune the
CLAUDE.md based on real customer interactions. This is not optional — the default
Claude personality is too generic for customer-facing support.

**What to watch for in the first hour**:

1. **Generic AI language** — if the bot says "I apologize for the inconvenience" or
   "Thank you for your patience", add explicit bans to CLAUDE.md
2. **Missing context** — if the bot asks questions it could answer by looking up the
   user's account, add mandatory context-gathering rules
3. **Wrong tone** — if customers write casually and the bot responds formally (or vice
   versa), add style mirroring rules
4. **Verbose responses** — if the bot writes paragraphs when one sentence would do,
   add length constraints
5. **"Let me check" syndrome** — if the bot announces what it's about to do instead
   of just doing it, add "fix first, report results" rules

**Real-world example of personality patch we applied in production**:

We added explicit good/bad examples to CLAUDE.md:

```markdown
## Response Examples

BAD (too corporate, too verbose):
"Hello! Thank you for reaching out. I'd be happy to help you with your issue.
Let me look into your account and check the status. I apologize for
any inconvenience you may be experiencing."

GOOD (human, direct, lowercase):
"checked your account — service srv-a3f8 is showing offline since 2h ago.
restarting it now. give it 30 seconds and try again."

BAD (announcing actions):
"I'm going to check your balance now."

GOOD (just do it):
"your balance is $12.50. last charge was $49 on apr 7."
```

Edit `/root/workspace/CLAUDE.md` directly — changes take effect on the next `claude -p`
invocation. No restart of any process needed.

---

## Troubleshooting

These are real issues we encountered during and after the migration.

| symptom | diagnosis | fix |
|---|---|---|
| `claude -p` returns "please run /login" | Missing or expired `.credentials.json` | Rewrite credentials file (Step 2 Option C) or re-copy from workstation |
| `claude -p` hangs forever | No `CLAUDE.md` in cwd (CLI is showing trust dialog) | Always `cd /root/workspace` before calling `claude -p`. The `-p` flag skips the trust dialog but only if CLAUDE.md exists |
| `claude -p` works manually but fails under PM2/systemd | `PATH` or `HOME` not set | Add `export PATH=/root/.local/bin:$PATH HOME=/root` at the top of every wrapper script |
| Telegram bridge running but no messages arrive | Stale offset from old OpenClaw gateway | Set `OFFSET=0` in the bridge script and restart. Or set to `-1` to get only the latest |
| Two bots replying to the same message | Old OpenClaw gateway still running, competing for updates | Stop and disable the old gateway completely. Check `pm2 list` AND `systemctl` |
| Telegram bridge receives messages but sends blank replies | Captured stderr instead of stdout | Use `RESPONSE=$(claude -p "$P" 2>/dev/null)` — redirect stderr to /dev/null |
| Bot gives generic AI responses | CLAUDE.md personality section too weak | Add explicit style rules, context-gathering requirements, and good/bad examples (Step 10) |
| Bot doesn't know user's history | No context-gathering before responding | Add mandatory tool calls (user lookup, ticket history) as the first step in CLAUDE.md |
| `Reached max turns` error | Default turn budget exceeded | Pass `--max-turns 5` (or higher) on the `claude -p` call |
| Costs appearing on Anthropic API dashboard | `ANTHROPIC_API_KEY` is set somewhere | `grep -r ANTHROPIC_API_KEY /etc/ /root/ 2>/dev/null` and remove every occurrence |
| `ANTHROPIC_AUTH_TOKEN` env var doesn't authenticate | OAuth tokens cannot be used as env vars | Use `~/.claude/.credentials.json` file instead (Step 2 Option C). Env var approach does not work. |
| NixOS rebuild fails "unit already exists" | Old gateway unit conflicts with new one | Set `systemd.services.openclaw-gateway.enable = false;` in NixOS config |
| Session memory leaks (disk fills with jsonl) | Bridge cleanup loop not running or too infrequent | Verify the `CYCLE % 30 == 0` block runs. Manual cleanup: `find /root/.claude/sessions -mtime +7 -delete` |
| Permission prompts on every tool call | Per-project settings missing | Write `/root/workspace/.claude/settings.json` with full permissions (Step 4) |
| MCP tools not available to claude -p | `.mcp.json` missing or not in cwd | Ensure `/root/workspace/.mcp.json` exists and `cd /root/workspace` before calling `claude -p` |

---

## Rollback

Every step is reversible without touching the skill scripts:

```bash
# 1. Stop the new processes
pm2 stop claude-telegram 2>/dev/null && pm2 delete claude-telegram 2>/dev/null
sudo systemctl disable --now claude-telegram 2>/dev/null

# 2. Re-enable the old gateway
pm2 start openclaw-gateway 2>/dev/null
sudo systemctl enable --now openclaw-gateway 2>/dev/null

# 3. (Optional) Remove the Claude Code runtime
sudo rm -rf /opt/claude-code /root/.claude /root/workspace
sudo rm /root/.local/bin/claude

# Skills and state at ~/.openclaw/* are UNTOUCHED.
# The original gateway picks up exactly where it left off.
```

---

## Rate Limit Budget

The Max plan uses a token bucket. Concrete budget reference on a busy bot:

| component | cadence | avg tokens/call | daily calls |
|---|---|---|---|
| Telegram free chat | user-driven | varies (500–5k) | 20–100 |
| Health check alerts | rare | 1–3k | 0–5 |
| Periodic reviews | every 2h | 8–15k | 12 |
| Pure workers | continuous | 0 (no LLM) | 0 |

This is well within the Max 20x budget. If you outgrow it, the same wrappers work
with `claude -p --model haiku` to drop to a cheaper model per call.

---

## Summary

The entire migration is:

1. Install `claude` CLI under `/opt/claude-code/`
2. Write `~/.claude/.credentials.json` with your OAuth token
3. Write `~/.claude/settings.json` with full permissions
4. Create `/root/workspace/CLAUDE.md` with your bot's brain
5. Create `/root/workspace/.mcp.json` with your tool servers
6. Write the Telegram bridge script
7. Start it with PM2 or systemd
8. Kill the old OpenClaw gateway
9. Test: send a Telegram message, verify response
10. Tune the CLAUDE.md personality based on real interactions

The skills stay. The state stays. The secrets stay. The database stays.
The billing changes from per-token to subscription. That's the whole migration.

**Time**: 20–40 minutes once you know the approach.
**This document**: So you know the approach.
