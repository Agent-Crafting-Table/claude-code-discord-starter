# Claude Code Discord Bot — Starter Kit

A ready-to-run Discord bot powered by Claude Code CLI. Uses your Claude Max subscription — no API key, no per-token billing.

**Coming from OpenClaw?** See [`MIGRATION.md`](./MIGRATION.md) — paste it into your existing agent and it will walk itself through the transfer.

---

## What's Included

| File | What it does |
|------|-------------|
| `Dockerfile` + `docker-compose.yml` | Container setup |
| `start.sh` | Launches Claude + cron runner + slash handler in tmux |
| `restart-loop.sh` | Keeps Claude running, restarts if it exits |
| `workspace/scripts/cron-runner.js` | Runs scheduled jobs from `crons/jobs.json` |
| `workspace/scripts/discord-slash-handler.js` | Handles `/status`, `/model`, `/cron`, `/ask` |
| `workspace/scripts/discord-slash-register.js` | Registers slash commands with Discord (run once) |
| `workspace/scripts/discord-post.js` | Helper to post messages to Discord from scripts/crons |
| `workspace/CLAUDE.md` | Your bot's identity — edit this |
| `workspace/crons/jobs.json` | Scheduled job definitions — edit this |
| `config/.env.example` | Environment variable template |

---

## Setup (5 steps)

### 1. Install Claude CLI and log in (on your local machine)

```bash
npm install -g @anthropic-ai/claude-code
claude   # opens browser to log in
```

You need a **Claude Max subscription** ($100/mo). Do not use an API key — it will bill per token.

### 2. Copy your auth

```bash
# Copy your claude auth into the starter kit
cp ~/.claude.json config/claude.json
```

### 3. Set up your Discord bot

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications)
2. New Application → add a Bot
3. Under **Bot → Privileged Gateway Intents**, enable all three intents
4. Copy your **Bot Token** and **Application ID**
5. Invite the bot to your server via OAuth2 URL Generator (scopes: `bot`, permissions: Send Messages, Read Message History, Add Reactions, Use Slash Commands)

### 4. Configure environment

```bash
cp config/.env.example config/.env
# Edit config/.env — fill in your DISCORD_BOT_TOKEN and DISCORD_APP_ID
```

### 5. Customize your bot

Edit `workspace/CLAUDE.md` — replace `[BotName]` and `[YourName]` with your bot's name and your name.

---

## Run with Docker (recommended)

```bash
# Build
docker-compose build --no-cache

# Start
docker-compose up -d

# Watch startup
docker logs claude-bot -f
```

Expected output:
```
[start] Claude Code Discord Bot starting...
[start] Claude is running.
[start] Starting cron runner...
[start] Starting slash command handler...
[start] All sessions running.
```

**On Unraid:** use `docker-compose` (with hyphen) — not `docker compose`.

### Attach to sessions

```bash
docker exec -it claude-bot tmux attach -t claude:0     # main Claude session
docker exec -it claude-bot tmux attach -t claude:cron  # cron runner
docker exec -it claude-bot tmux attach -t claude:slash # slash handler
# Detach without stopping: Ctrl+B then D
```

---

## Run without Docker (Mac or Linux VPS)

```bash
# Install dependencies
cd workspace && npm install && cd ..

# Load env and start
set -a && source config/.env && set +a
export WORKSPACE_DIR="$(pwd)/workspace"

# Copy auth to home dir (where Claude looks for it)
cp config/claude.json ~/.claude.json

# Terminal 1 — Claude
bash restart-loop.sh

# Terminal 2 — Cron runner
node workspace/scripts/cron-runner.js

# Terminal 3 — Slash handler
node workspace/scripts/discord-slash-handler.js
```

For always-on on a VPS, wrap in tmux:
```bash
tmux new-session -d -s claude "bash restart-loop.sh"
tmux new-window -t claude -n cron "node workspace/scripts/cron-runner.js"
tmux new-window -t claude -n slash "node workspace/scripts/discord-slash-handler.js"
```

---

## Connect Discord Channels

After the bot is running, attach to the Claude session and run:

```
/discord:access opt-in
```

Copy the pairing code. In Discord, DM your bot:
```
/discord:access pair <code>
```

Allow your user ID to DM the bot:
```
/discord:access allow YOUR_DISCORD_USER_ID
```
(Right-click your name → Copy User ID. Enable Developer Mode in Discord settings if you don't see this.)

Allow a channel without needing @mention:
```
/discord:access channel #your-channel --no-require-mention
```

### Enable the 👀 ack reaction

So you immediately know your bot saw your message:

```bash
# Run inside the container (or adjust path for non-Docker)
docker exec claude-bot node -e "
const fs = require('fs');
const f = '/home/node/.claude/channels/discord/access.json';
const data = JSON.parse(fs.readFileSync(f, 'utf8'));
data.ackReaction = '👀';
fs.writeFileSync(f, JSON.stringify(data, null, 2) + '\n');
console.log('Done');
"
```

---

## Register Slash Commands

Run once after the bot is running:

```bash
docker exec claude-bot node /workspace/scripts/discord-slash-register.js
```

Global commands take up to 1 hour to appear in Discord. For instant testing, see the comment in `discord-slash-register.js` about guild commands.

Available commands after registering:
- `/status` — health check
- `/model [opus|sonnet|haiku]` — show or switch model
- `/compact` — compact Claude's context
- `/cron list` — see scheduled jobs
- `/cron run <name>` — trigger a job manually
- `/cron logs <name>` — view recent job output
- `/ask <message>` — send anything to your bot

---

## Add Scheduled Jobs

Edit `workspace/crons/jobs.json`. Each job:

```json
{
  "id": "my-job",
  "name": "My Job",
  "schedule": "0 9 * * 1-5",
  "tz": "America/Denver",
  "enabled": true,
  "timeoutSeconds": 120,
  "model": "sonnet",
  "discordChannel": "YOUR_CHANNEL_ID",
  "announceResult": true,
  "message": "The prompt Claude receives. Write it like you're asking Claude to do something."
}
```

`announceResult: true` posts the job output to `discordChannel` when it finishes.

Crons always use `sonnet` by default. Override per-job with `"model": "opus"`.

For long jobs, add this to your message: _"Use a subagent for this task so the main session stays responsive."_ Claude Code supports parallel subagents natively.

---

## Troubleshooting

**Bot goes offline repeatedly**
```bash
docker logs claude-bot --tail 50
docker exec -it claude-bot tmux attach -t claude:0
```
Look for auth errors → re-copy `~/.claude.json` to `config/claude.json` and restart.

**Bot online but not responding to messages**
- Check the channel is opted in: inside Claude run `/discord:access status`
- Check Message Content Intent is on in the Discord Developer Portal

**Slash commands not showing**
- Re-run `discord-slash-register.js` and wait up to 1 hour for global propagation

**`ANTHROPIC_API_KEY` was accidentally set**
```bash
docker exec claude-bot env | grep ANTHROPIC_API_KEY  # should return nothing
# If it shows a value, remove it from config/.env and restart
```

**Cron runner window missing**
```bash
docker exec claude-bot tmux list-windows -t claude
# If 'cron' is missing:
docker exec claude-bot tmux new-window -t claude -n cron \
  "while true; do node /workspace/scripts/cron-runner.js 2>&1; sleep 10; done"
```

---

## File Structure

```
claude-code-starter/
├── Dockerfile
├── docker-compose.yml
├── start.sh                        # container entrypoint
├── restart-loop.sh                 # keeps Claude running
├── .gitignore
├── config/
│   ├── .env.example                # copy to .env and fill in
│   └── claude.json                 # copy from ~/.claude.json (gitignored)
├── claude-data/                    # Claude plugin state (auto-created, gitignored)
└── workspace/
    ├── CLAUDE.md                   # bot identity — edit this
    ├── package.json
    ├── crons/
    │   ├── jobs.json               # scheduled jobs — edit this
    │   └── logs/                   # job output logs (gitignored)
    ├── data/
    │   └── current-model.json      # active model preference
    ├── memory/                     # daily notes, threads (gitignored)
    └── scripts/
        ├── cron-runner.js
        ├── discord-post.js
        ├── discord-slash-handler.js
        └── discord-slash-register.js
```
