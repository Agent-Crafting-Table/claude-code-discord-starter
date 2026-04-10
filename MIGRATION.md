# OpenClaw → Claude Code Migration Guide

Move your AI Discord bot from OpenClaw to Claude Code CLI. No API key required — uses your Claude Max subscription.

Three deployment options covered:
- **Option A** — Run locally on your Mac (simplest, bot goes offline when Mac sleeps)
- **Option B** — Run on a VPS/Linux server (always on, no special hardware)
- **Option C** — Run on Unraid (always on, if you already have Unraid)

Pick one. The setup is nearly identical across all three — the only difference is where the files live and how the process stays running.

---

## Before You Start — Things You Need

- [ ] **Claude Max subscription** — the $100/mo plan or higher. Required. This is what lets you run Claude without an API key. Do not use an API key — it will bill you per token and get expensive fast.
- [ ] **Discord bot** — create one at [discord.com/developers/applications](https://discord.com/developers/applications). You need the bot token and application ID.
- [ ] **`claude` CLI installed locally** on your Mac — `npm install -g @anthropic-ai/claude-code` — even if you're deploying to a server, you need this locally first to log in.
- [ ] **Node.js 18+** on whatever machine is running the bot.

---

## Step 1 — Log In to Claude Locally (Do This First, On Your Mac)

```bash
claude
```

Follow the browser login. This saves your auth to `~/.claude.json`. You only need to do this once.

After login, grab your OAuth token:
```bash
cat ~/.claude.json
```

Copy the whole file. You'll need it on whichever machine runs your bot.

**The golden rule: never set `ANTHROPIC_API_KEY`.** If that env var is present, Claude ignores your subscription and bills your API account instead. Leave it out entirely.

---

## Step 2 — Create Your Discord Bot

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications)
2. Click **New Application** → give it a name → go to the **Bot** tab
3. Click **Add Bot** → copy the **Token** (you'll need this later)
4. Copy the **Application ID** from the General Information page
5. Under **Bot → Privileged Gateway Intents**, turn on all three:
   - Presence Intent
   - Server Members Intent
   - Message Content Intent
6. Under **OAuth2 → URL Generator**:
   - Scopes: `bot`
   - Bot Permissions: `Send Messages`, `Read Message History`, `Add Reactions`, `Use Slash Commands`
7. Copy the generated URL → open it in a browser → add the bot to your server

---

## Step 3 — Create the Bot Workspace

Pick a folder where your bot will live. Examples:
- Mac: `~/claude-bot`
- VPS: `/home/youruser/claude-bot`
- Unraid: `/mnt/user/appdata/claude-bot/workspace`

```bash
WORKSPACE=~/claude-bot   # change this to your path

mkdir -p $WORKSPACE/{memory,scripts,crons/logs,data}
```

Copy your Claude auth into the workspace:
```bash
cp ~/.claude.json $WORKSPACE/claude.json
```

---

## Step 4 — Create Your Environment File

Create `$WORKSPACE/.env`:

```ini
# Discord bot credentials
DISCORD_BOT_TOKEN=your_bot_token_here
DISCORD_APP_ID=your_application_id_here

# DO NOT add ANTHROPIC_API_KEY — leave it out entirely
```

Lock it down:
```bash
chmod 600 $WORKSPACE/.env
```

---

## Step 5 — Create the Bot Identity File

Create `$WORKSPACE/CLAUDE.md` — Claude reads this at every session start:

```markdown
# CLAUDE.md

You are **[YourBotName]** — [YourName]'s AI assistant running on Claude Code.

## On Every Session Start
1. Read `memory/active-threads.md` if it exists — what's in progress
2. Read today's daily note `memory/YYYY-MM-DD.md` if it exists

## How You Behave
- Do the work first, talk about it after. Don't over-explain.
- For tasks that take more than ~15 seconds, post a ✅ Done message when finished so the user knows you're done.
- Write brief notes to `memory/YYYY-MM-DD.md` after completing tasks so you remember them next session.
- Match the user's energy — relaxed, no corporate speak.

## Action Policy
- Reading files, running scripts, checking things: do it freely.
- Sending messages, emails, posting anything external: ask first unless it's an established workflow.
- Anything destructive (deleting, overwriting): always warn first.

## Formatting for Discord
- No markdown tables — use bullet lists instead.
- Wrap URLs in <> to suppress Discord link previews.
- Keep responses concise.
```

---

## Step 6 — Create the Cron Runner Script

This script handles scheduled jobs. Create `$WORKSPACE/scripts/cron-runner.js`:

```js
#!/usr/bin/env node
'use strict';

const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');
const cron = require('node-cron');

const WORKSPACE = process.env.WORKSPACE_DIR || path.join(__dirname, '..');
const JOBS_FILE = path.join(WORKSPACE, 'crons', 'jobs.json');
const LOG_DIR   = path.join(WORKSPACE, 'crons', 'logs');

fs.mkdirSync(LOG_DIR, { recursive: true });

function loadJobs() {
  try { return JSON.parse(fs.readFileSync(JOBS_FILE, 'utf8')).jobs || []; }
  catch { return []; }
}

function log(name, msg) {
  console.log(`[${new Date().toISOString()}] [${name}] ${msg}`);
}

function runJob(job) {
  log(job.name, 'Starting');
  const logFile = path.join(LOG_DIR, `${job.id}.log`);
  const out = fs.openSync(logFile, 'a');

  const child = spawn(
    'claude',
    ['--dangerously-skip-permissions', '--model', job.model || 'sonnet', '-p', job.message],
    { cwd: WORKSPACE, stdio: ['ignore', out, out], env: { ...process.env } }
  );

  const timeout = setTimeout(() => {
    child.kill('SIGTERM');
    log(job.name, 'Timed out');
  }, (job.timeoutSeconds || 300) * 1000);

  child.on('close', code => {
    clearTimeout(timeout);
    fs.closeSync(out);
    log(job.name, `Done (exit ${code})`);
  });
}

function start() {
  const jobs = loadJobs().filter(j => j.enabled);
  log('cron-runner', `Loaded ${jobs.length} enabled job(s)`);

  for (const job of jobs) {
    log(job.name, `Scheduled: ${job.schedule}`);
    cron.schedule(job.schedule, () => runJob(job), {
      timezone: process.env.CRON_TZ || 'America/Denver'
    });
  }
}

start();
```

Install the cron dependency:
```bash
cd $WORKSPACE && npm init -y && npm install node-cron
```

Create `$WORKSPACE/crons/jobs.json` with your scheduled tasks:
```json
{
  "jobs": [
    {
      "id": "daily-status",
      "name": "Daily Status",
      "schedule": "0 9 * * *",
      "enabled": true,
      "timeoutSeconds": 120,
      "model": "sonnet",
      "message": "Post a brief good morning status to Discord. Check if anything needs attention today. Post result to Discord channel YOUR_CHANNEL_ID using: node /path/to/workspace/scripts/discord-post.js YOUR_CHANNEL_ID \"message here\""
    }
  ]
}
```

---

## Step 7 — Create the Discord Post Helper

Create `$WORKSPACE/scripts/discord-post.js` — lets Claude and crons post to Discord directly:

```js
#!/usr/bin/env node
'use strict';

const https = require('https');

async function postToDiscord(channelId, message) {
  const token = process.env.DISCORD_BOT_TOKEN;
  if (!token) throw new Error('DISCORD_BOT_TOKEN not set');

  const content = message.length > 2000 ? message.slice(0, 1997) + '...' : message;
  const body = JSON.stringify({ content });

  return new Promise((resolve, reject) => {
    const req = https.request({
      hostname: 'discord.com',
      path: `/api/v10/channels/${channelId}/messages`,
      method: 'POST',
      headers: {
        'Authorization': `Bot ${token}`,
        'Content-Type': 'application/json',
        'Content-Length': Buffer.byteLength(body),
      },
    }, res => {
      let data = '';
      res.on('data', chunk => (data += chunk));
      res.on('end', () => {
        if (res.statusCode >= 200 && res.statusCode < 300) resolve(JSON.parse(data));
        else reject(new Error(`Discord API ${res.statusCode}: ${data}`));
      });
    });
    req.on('error', reject);
    req.write(body);
    req.end();
  });
}

if (require.main === module) {
  const [,, channelId, ...parts] = process.argv;
  const message = parts.join(' ');
  if (!channelId || !message) {
    console.error('Usage: node discord-post.js <channelId> <message>');
    process.exit(1);
  }
  postToDiscord(channelId, message)
    .then(() => process.exit(0))
    .catch(err => { console.error(err.message); process.exit(1); });
}

module.exports = { postToDiscord };
```

---

## Step 8 — Enable the 👀 Ack Reaction

After you've connected Discord (Step 10 below), run this once to make the bot react immediately when it sees your message — so you know it's working even before it replies:

```bash
# Run inside the workspace where claude-data lives
node -e "
const fs = require('fs');
const os = require('os');
const f = os.homedir() + '/.claude/channels/discord/access.json';
const data = JSON.parse(fs.readFileSync(f, 'utf8'));
data.ackReaction = '👀';
fs.writeFileSync(f, JSON.stringify(data, null, 2) + '\n');
console.log('Done — bot will now react 👀 when it sees your messages');
"
```

---

## Option A — Run Locally on Your Mac

Simplest setup. Bot goes offline when your Mac sleeps or you close the terminal. Good for testing.

### Start the bot

Load your env and start Claude with Discord:

```bash
cd $WORKSPACE
set -a && source .env && set +a   # load env vars

# Start Claude connected to Discord
claude --dangerously-skip-permissions --model sonnet \
  --channels plugin:discord@claude-plugins-official
```

Claude will stay running and respond to Discord messages. Open a second terminal tab for the cron runner:

```bash
cd $WORKSPACE
set -a && source .env && set +a
node scripts/cron-runner.js
```

### Keep it running with a simple restart script

Create `$WORKSPACE/run.sh`:
```bash
#!/bin/bash
cd "$(dirname "$0")"
set -a && source .env && set +a

while true; do
  echo "[$(date)] Starting Claude..."
  claude --dangerously-skip-permissions --model sonnet \
    --channels plugin:discord@claude-plugins-official
  echo "[$(date)] Claude exited. Restarting in 10s..."
  sleep 10
done
```

```bash
chmod +x $WORKSPACE/run.sh
./run.sh
```

---

## Option B — Run on a VPS / Linux Server

Always on. Works on any Linux VPS (DigitalOcean, Linode, Hetzner, etc.). Recommended over local if you want reliability.

### Install Node.js on the VPS

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version   # should say v22.x
```

### Install Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
```

### Copy your auth and workspace to the VPS

From your Mac:
```bash
VPS=user@your-vps-ip
WORKSPACE=~/claude-bot

# Copy workspace files
scp -r $WORKSPACE $VPS:~/claude-bot

# Copy your claude auth
scp ~/.claude.json $VPS:~/.claude.json
```

### Install tmux (keeps sessions alive when you disconnect)

```bash
sudo apt-get install -y tmux
```

### Start with tmux so it survives SSH disconnects

```bash
# On the VPS
tmux new-session -d -s claude "bash ~/claude-bot/run.sh"
tmux new-window -t claude -n cron "cd ~/claude-bot && set -a && source .env && set +a && node scripts/cron-runner.js"

# Attach to watch it:
tmux attach -t claude
# Detach without stopping: Ctrl+B then D
```

### Make it restart on VPS reboot with systemd

Create `/etc/systemd/system/claude-bot.service`:

```ini
[Unit]
Description=Claude Code Discord Bot
After=network.target

[Service]
Type=forking
User=youruser
WorkingDirectory=/home/youruser/claude-bot
EnvironmentFile=/home/youruser/claude-bot/.env
ExecStart=/usr/bin/tmux new-session -d -s claude "bash /home/youruser/claude-bot/run.sh"
ExecStop=/usr/bin/tmux kill-session -t claude
RemainAfterExit=yes
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable claude-bot
sudo systemctl start claude-bot
sudo systemctl status claude-bot
```

Check logs:
```bash
tmux attach -t claude         # main Claude session
# or
journalctl -u claude-bot -f   # systemd logs
```

---

## Option C — Run on Unraid (Docker)

For Unraid users who already have Docker set up. Creates a proper container that survives reboots.

### Directory structure on Unraid

```bash
APPDATA=/mnt/user/appdata/claude-bot

mkdir -p $APPDATA/{workspace/{memory,scripts,crons/logs,data},config,claude-data}

# Copy your claude auth
cp ~/.claude.json $APPDATA/config/claude.json

# Create workspace files same as Steps 3–7 above but under:
# $APPDATA/workspace/ instead of ~/claude-bot/
```

### Dockerfile

Create `/mnt/user/appdata/claude-bot/Dockerfile`:

```dockerfile
FROM node:22-bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates tmux git openssh-client \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g @anthropic-ai/claude-code

RUN mkdir -p /home/node/.claude /workspace /config && \
    chown -R node:node /home/node /workspace

WORKDIR /workspace
USER node
ENV HOME=/home/node

COPY --chown=node:node start.sh /start.sh
COPY --chown=node:node restart-loop.sh /restart-loop.sh
RUN chmod +x /start.sh /restart-loop.sh

CMD ["/start.sh"]
```

### docker-compose.yml

Create `/mnt/user/appdata/claude-bot/docker-compose.yml`:

```yaml
version: "3.8"

services:
  claude-bot:
    build: .
    container_name: claude-bot
    restart: always
    volumes:
      - /mnt/user/appdata/claude-bot/config:/config:ro
      - /mnt/user/appdata/claude-bot/workspace:/workspace
      - /mnt/user/appdata/claude-bot/claude-data:/home/node/.claude
      - /mnt/user/appdata/claude-bot/start.sh:/start.sh:ro
      - /mnt/user/appdata/claude-bot/restart-loop.sh:/restart-loop.sh:ro
    env_file:
      - /mnt/user/appdata/claude-bot/config/.env
    environment:
      - HOME=/home/node
```

### start.sh

Create `/mnt/user/appdata/claude-bot/start.sh`:

```bash
#!/bin/bash
set -e

cp /config/claude.json /home/node/.claude.json
chmod 600 /home/node/.claude.json

tmux new-session -d -s claude "bash /restart-loop.sh"

# Auto-accept Claude's startup prompts
for i in $(seq 1 20); do
  sleep 2
  OUTPUT=$(tmux capture-pane -t claude -p 2>/dev/null || true)
  if echo "$OUTPUT" | grep -q "Is this a project you created"; then
    tmux send-keys -t claude '' Enter
  elif echo "$OUTPUT" | grep -q "Yes, I accept"; then
    tmux send-keys -t claude j ''
    sleep 0.5
    tmux send-keys -t claude '' Enter
  elif echo "$OUTPUT" | grep -q "bypass permissions on\|Listening for channel"; then
    echo "Claude is running."
    break
  fi
done

# Cron runner with auto-restart
tmux new-window -t claude -n cron "while true; do node /workspace/scripts/cron-runner.js 2>&1 | tee -a /workspace/crons/logs/cron-runner.log; sleep 10; done"

tail -f /dev/null
```

### restart-loop.sh

Create `/mnt/user/appdata/claude-bot/restart-loop.sh`:

```bash
#!/bin/bash
MODEL_STATE="/workspace/data/current-model.json"

get_model() {
  if [ -f "$MODEL_STATE" ]; then
    node -e "try{process.stdout.write(JSON.parse(require('fs').readFileSync('$MODEL_STATE','utf8')).model||'sonnet')}catch(e){process.stdout.write('sonnet')}"
  else
    echo "sonnet"
  fi
}

while true; do
  MODEL=$(get_model)
  echo "[$(date)] Starting Claude with model: $MODEL"
  claude --dangerously-skip-permissions --model "$MODEL" \
    --channels plugin:discord@claude-plugins-official
  echo "[$(date)] Claude exited. Restarting in 10s..."
  sleep 10
done
```

### Build and run (always hyphenated on Unraid — `docker-compose`, not `docker compose`)

```bash
cd /mnt/user/appdata/claude-bot
docker-compose build --no-cache
docker-compose up -d
docker logs claude-bot -f
```

### Attach to the running session

```bash
docker exec -it claude-bot tmux attach -t claude:0   # main Claude
docker exec -it claude-bot tmux attach -t claude:cron # cron runner
```

---

## Step 10 — Connect Discord Channels (All Options)

Once Claude is running, connect it to your Discord channels.

**Attach to the Claude session:**
- Mac/VPS: `tmux attach -t claude` (then select window 0)
- Unraid: `docker exec -it claude-bot tmux attach -t claude:0`

**Inside Claude, run:**
```
/discord:access opt-in
```

It will give you a pairing code. In Discord, DM your bot:
```
/discord:access pair [the-code]
```

**Allow your Discord user ID to message the bot:**
```
/discord:access allow YOUR_DISCORD_USER_ID
```
(Right-click your name in Discord → Copy User ID. Enable Developer Mode in Discord settings if you don't see this option.)

**Allow a channel without requiring @mention:**
```
/discord:access channel #your-channel-name --no-require-mention
```
Repeat for each channel you want Herc to watch.

**Then run the 👀 ack reaction setup from Step 8.**

---

## Troubleshooting

### Bot starts but goes offline after a minute
The restart loop isn't set up — Claude exited and nothing relaunched it. Make sure you're using `run.sh` or `restart-loop.sh` instead of running `claude` directly.

### Claude session keeps restarting every 10 seconds
Auth issue. Check:
```bash
# Mac/VPS
cat ~/.claude.json   # should have a valid token

# If token expired: run `claude` locally on your Mac, log in again, recopy ~/.claude.json to the server
```

### Bot is online in Discord but doesn't respond to messages
1. Check that the channel is opted in — inside Claude run `/discord:access status`
2. Check that `requireMention` is false for your channel — look at `~/.claude/channels/discord/access.json`
3. Confirm Message Content Intent is on in the Discord Developer Portal (Step 2)

### Cron jobs not firing
Check the cron runner is actually running:
```bash
# Mac/VPS
tmux list-windows -t claude    # should see 'cron' window

# Unraid
docker exec claude-bot tmux list-windows -t claude
```

If missing, start it manually:
```bash
# Mac/VPS
tmux new-window -t claude -n cron "cd ~/claude-bot && set -a && source .env && set +a && node scripts/cron-runner.js"
```

### `ANTHROPIC_API_KEY` was accidentally set
Remove it from your `.env` file and restart. Check with:
```bash
env | grep ANTHROPIC_API_KEY   # should return nothing
```

### `docker compose` not found / fails on Unraid
Use `docker-compose` (with a hyphen). Unraid ships with docker-compose v1 and doesn't have the v2 `docker compose` plugin.

---

## Key Differences from OpenClaw

| Thing | OpenClaw | Claude Code |
|---|---|---|
| Runtime | `openclaw` npm package | `claude` CLI |
| Identity/config | `config.yml` | `CLAUDE.md` in workspace |
| Scheduled jobs | Built-in lobster pipeline | `cron-runner.js` + `jobs.json` |
| Skills/tools | Plugin system | Scripts in `scripts/` folder |
| Discord posting | Built-in tool | `discord-post.js` helper script |
| Health check | HTTP `/health` endpoint | `tmux has-session` |
| Billing | API key (per-token) | Claude Max subscription (flat rate) |
| "I see you" signal | Streams output live | 👀 reaction + ✅ done message |
| Long task handling | Streams thoughts in real time | Subagents run in parallel, main session stays free |
