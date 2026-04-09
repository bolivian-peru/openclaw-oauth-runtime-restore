---
name: clawswap
description: >
  Migrate an OpenClaw deployment to Claude Code native runtime.
  Swaps API-credit billing for Max-plan OAuth. Non-destructive —
  keeps all skills, state, and configs. Generates shell wrappers,
  systemd units, and an optional Telegram bridge.
trigger: |
  Use when the user says: /clawswap, migrate openclaw, swap runtime,
  move off API credits, convert to claude code, or clawback.
allowed-tools:
  - "Bash(*)"
  - "Read(*)"
  - "Write(*)"
  - "Edit(*)"
requires:
  bins: [bash, curl, node, npm, jq, systemctl]
  runtime:
    - "Anthropic Claude Max plan account (for OAuth token)"
    - "Existing OpenClaw deployment under ~/.openclaw/"
---

# ClawSwap — OpenClaw → Claude Code Runtime Migration

One master skill. Swaps the runtime, keeps the claws. Four hours, tested on
[os-moda](https://github.com/bolivian-peru/os-moda/), works on any OpenClaw setup.

## Rules

- NEVER delete anything under `~/.openclaw/` — the migration is non-destructive
- NEVER set `ANTHROPIC_API_KEY` anywhere — OAuth must be the only auth method
- Always confirm before writing systemd units or modifying NixOS config
- Ask which skills exist before generating wrappers

## Architecture

```
                    ┌──────────────────────────────┐
                    │  /root/.claude/               │
                    │   .credentials.json (OAuth)   │
                    │   settings.json (perms)       │
                    │   projects/-root-workspace/   │  ← session jsonl per conv
                    └──────────────┬───────────────┘
                                   │
   /root/workspace/  (always cwd)  │
   ├─ CLAUDE.md       (auto-loaded)│
   ├─ skills/         (legacy)     │
   └─ .openclaw → /root/.openclaw  │
                                   │
               ┌───────────────────┴────────────────┐
               │                                    │
    /opt/claude-code/                      /root/.openclaw/skills/
    ├─ heartbeat.sh   ───┐                 ├─ skill-a/
    ├─ mm-review.sh   ───┤  shell wrappers ├─ skill-b/
    ├─ rewards-*.sh   ───┤  build prompts  └─ skill-c/
    └─ telegram-bridge.sh│  and pipe into
              ▼          │    claude -p
    @anthropic-ai/claude-code  ──→  Anthropic OAuth (Max plan quota)
                                   │
                                   ▼
                        systemd timers / services
```

## Step 1 — Inventory

Scan the existing OpenClaw setup before touching anything:

```bash
# invocation points
systemctl list-unit-files --no-pager | grep -Ei 'openclaw|agentd|osmoda'
crontab -l 2>/dev/null
ls -la ~/.openclaw/skills/

# state + config
ls ~/.openclaw/*/state/ ~/.openclaw/*/config/ 2>/dev/null
find ~/.openclaw -name 'settings.json' -maxdepth 4

# secrets (never delete these)
find ~/.openclaw -name '*.key' -o -name 'secrets*' -o -name '.env*' 2>/dev/null
ls /var/lib/osmoda/secrets/ 2>/dev/null
```

Write results to a scratch file. Report what you find. Confirm before proceeding.

## Step 2 — Install Claude Code

```bash
node --version    # must be >= v20

sudo mkdir -p /opt/claude-code
cd /opt/claude-code

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
sudo mkdir -p /root/.local/bin
sudo ln -sf /opt/claude-code/node_modules/.bin/claude /root/.local/bin/claude
/root/.local/bin/claude --version
```

## Step 3 — OAuth Token (Max plan auth)

This is the key step. Claude Code stores OAuth material in `~/.claude/.credentials.json`
(mode 600). The CLI refreshes the access token automatically — no cron needed.

### Option A — Interactive (has browser)

```bash
HOME=/root /root/.local/bin/claude    # then type /login in the REPL
```

### Option B — Headless VPS (most common)

1. Run `claude /login` on a workstation with a browser
2. Authenticate with the Max plan account
3. Copy to VPS:

```bash
scp ~/.claude/.credentials.json vps:/root/.claude/.credentials.json
ssh vps "chmod 600 /root/.claude/.credentials.json && chown root:root /root/.claude/.credentials.json"
```

### Verify

```bash
jq .claudeAiOauth.subscriptionType /root/.claude/.credentials.json
# → "max"
jq .claudeAiOauth.rateLimitTier /root/.claude/.credentials.json
# → "default_claude_max_20x" (or _5x)
```

Expected credential shape (tokens redacted):

```json
{
  "claudeAiOauth": {
    "accessToken":  "sk-ant-oat01-…",
    "refreshToken": "sk-ant-ort01-…",
    "expiresAt":    1775774290080,
    "scopes": ["user:inference", "user:profile", "user:sessions:claude_code"],
    "subscriptionType": "max",
    "rateLimitTier":    "default_claude_max_20x"
  }
}
```

**WARNING**: Do NOT set `ANTHROPIC_API_KEY` anywhere. If both exist, the API key
wins and you fall back to credit billing. Audit all systemd units and shell
profiles for stray exports.

## Step 4 — Workspace + CLAUDE.md

```bash
sudo mkdir -p /root/workspace
cd /root/workspace
sudo ln -sf /root/.openclaw .openclaw
```

Generate a CLAUDE.md that lists:
- Host info (distro, services running)
- Every skill from Step 1 (name, script paths, config paths, state paths)
- Periodic task schedule (heartbeat every 5m, review every 2h, etc.)
- Safety rules (never modify system config without approval, never push without approval)

Claude Code auto-loads `CLAUDE.md` from cwd on every invocation. Sessions land
in `~/.claude/projects/-root-workspace/<session-id>.jsonl`.

## Step 5 — Permissions

```bash
sudo mkdir -p /root/.claude /root/workspace/.claude

# Write both global and per-project settings (same content):
for f in /root/.claude/settings.json /root/workspace/.claude/settings.json; do
  sudo tee "$f" >/dev/null <<'JSON'
{
  "permissions": {
    "allow": ["Bash(*)","Read(*)","Write(*)","Edit(*)","Glob(*)","Grep(*)","Agent(*)"],
    "deny": []
  },
  "enableAllProjectMcpServers": true
}
JSON
done
```

## Step 6 — Convert Skill Wrappers

For each `openclaw <skill>` from the inventory, generate a wrapper in `/opt/claude-code/`.
Three patterns cover ~95% of cases:

### Pattern A — Health check (no LLM unless alert)

```bash
#!/usr/bin/env bash
export PATH=/root/.local/bin:/run/current-system/sw/bin:$PATH HOME=/root
LOG="/root/.openclaw/<skill>/state/heartbeat.log"

cpu=$(awk '{print $1}' /proc/loadavg)
mem=$(free | awk '/Mem:/{printf "%.0f", $3/$2*100}')
disk=$(df / | awk 'NR==2{print $5}' | tr -d %)
down=""
for svc in <svc-1> <svc-2>; do
    systemctl is-active "$svc" >/dev/null 2>&1 || down="$down $svc"
done
echo "$(date -u +%FT%TZ) load=$cpu mem=${mem}% disk=${disk}%${down:+ DOWN:$down}" >> "$LOG"

if (( mem > 85 )) || (( disk > 90 )) || [[ -n "$down" ]]; then
    claude -p "ALERT: load=$cpu mem=${mem}% disk=${disk}% down=$down. Investigate." \
        --max-turns 3 >> "$LOG" 2>&1
fi
```

### Pattern B — Periodic LLM review with Telegram output

```bash
#!/usr/bin/env bash
export PATH=/root/.local/bin:/run/current-system/sw/bin:$PATH HOME=/root
STATE_DIR="/root/.openclaw/<skill>/state"
BOT_TOKEN=$(cat /var/lib/osmoda/secrets/telegram-bot-token 2>/dev/null)
CHAT_ID="<your-chat-id>"

send_tg() {
    [[ -z "$BOT_TOKEN" ]] && return
    curl -s "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        -d "chat_id=$CHAT_ID" --data-urlencode "text=$1" >/dev/null 2>&1
}

# Gather state
DATA=$(cat "$STATE_DIR"/*.json 2>/dev/null)
RECENT=$(tail -25 "$STATE_DIR/run.log" 2>/dev/null)

PROMPT="Review this state and report.
DATA: $DATA
LOG: $RECENT
Emit a ---TELEGRAM--- block with 3-5 line summary, then ---END---"

cd /root/workspace
RESPONSE=$(claude -p "$PROMPT" 2>/dev/null)
echo "$(date -u +%FT%TZ)" >> "$STATE_DIR/review.log"
echo "$RESPONSE" >> "$STATE_DIR/review.log"

TG_MSG=$(echo "$RESPONSE" | sed -n '/---TELEGRAM---/,/---END---/p' | grep -v '^---')
[[ -n "$TG_MSG" ]] && send_tg "$TG_MSG"
```

### Pattern C — Pure worker (no LLM)

```bash
#!/usr/bin/env bash
export PATH=/root/.local/bin:/run/current-system/sw/bin:$PATH HOME=/root
cd /root/.openclaw/skills/<skill>
exec node scripts/<worker>.mjs
```

Ask the user which pattern each skill needs. Generate the wrapper scripts.

## Step 7 — Systemd

Detect NixOS (`test -f /etc/nixos/configuration.nix`) or standard systemd.

### NixOS — generate `/etc/nixos/claude-code.nix`

```nix
{ pkgs, ... }:
{
  systemd.services.claude-heartbeat = {
    description = "Claude Code Heartbeat";
    serviceConfig = {
      Type = "oneshot";
      ExecStart = "/opt/claude-code/heartbeat.sh";
      Environment = [ "PATH=/root/.local/bin:/run/current-system/sw/bin" "HOME=/root" ];
    };
  };
  systemd.timers.claude-heartbeat = {
    wantedBy = [ "timers.target" ];
    timerConfig = { OnCalendar = "*:0/5"; Persistent = true; };
  };

  # repeat for each wrapper...

  systemd.services.osmoda-gateway.enable = false;  # kill the old gateway
}
```

Import from `configuration.nix`, then `sudo nixos-rebuild switch`.

### Standard Linux

```bash
# drop .service + .timer files in /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now claude-heartbeat.timer
sudo systemctl enable --now claude-mm-review.timer
sudo systemctl disable --now osmoda-gateway
```

## Step 8 — Telegram Bridge (optional)

Long-running bash daemon that polls `getUpdates` and routes to `claude -p --resume`.

Key mechanism:
```bash
# per-conversation session persistence
RESPONSE=$(cd /root/workspace && claude -p "$TEXT" --resume "$SESSION_ID" 2>/dev/null)
LATEST=$(ls -t /root/.claude/sessions/ 2>/dev/null | head -1)
[[ -n "$LATEST" ]] && save_session_id "$ACTIVE" "$LATEST"
```

Slash commands: `/menu /new /list /switch /current /delete /status`

Requires: `BOT_TOKEN` in `/var/lib/osmoda/secrets/telegram-bot-token` and `CHAT_ID`
hardcoded in the bridge script.

## Step 9 — Verify

```bash
# CLI installed + authenticated
/root/.local/bin/claude --version
jq .claudeAiOauth.subscriptionType /root/.claude/.credentials.json

# Smoke test (must not prompt for browser)
HOME=/root /root/.local/bin/claude -p "say only the word: pong" 2>&1 | head

# Auto-context loading
HOME=/root /root/.local/bin/claude -p "what hostname am I on? one line" 2>&1

# Tool permissions (no gate)
HOME=/root /root/.local/bin/claude -p "run: echo hello && date -u" 2>&1

# Systemd units active
systemctl list-timers --all --no-pager | grep claude
systemctl status claude-telegram --no-pager | head -5

# Old gateway dead
systemctl status osmoda-gateway --no-pager | head -3
```

## Step 10 — Done

Tell the user: "Runtime swapped. Your claws are back. $0 extra per token."

## Troubleshooting

| symptom | fix |
|---|---|
| `please run /login` | missing or expired `.credentials.json` — re-copy from workstation |
| `claude -p` hangs | no `CLAUDE.md` in cwd — `cd /root/workspace` first |
| `Reached max turns` | pass `--max-turns 5` (or higher) |
| telegram replies blank | use `RESPONSE=$(claude -p "$P" 2>/dev/null)` — suppress stderr |
| costs on API dashboard | `ANTHROPIC_API_KEY` set somewhere — `grep -r ANTHROPIC_API_KEY /etc/ /root/` |
| NixOS "unit already exists" | set `systemd.services.osmoda-gateway.enable = false;` |
| two bots replying | old `openclaw-gateway` still running — disable it |
| permission prompts | per-project settings missing — check Step 5 |

## Rollback

Every step is reversible without touching skill scripts:

```bash
sudo systemctl disable --now claude-telegram \
    claude-heartbeat.timer claude-mm-review.timer 2>/dev/null
sudo systemctl enable --now osmoda-gateway
# ~/.openclaw/* is untouched. original gateway picks up where it left off.
```
