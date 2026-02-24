# OpenClaw on Railway: Lead Triangulation Setup Guide

Complete walkthrough — from deploying OpenClaw on Railway to running an automated daily analysis that cross-references HubSpot, Fullstory, and Gong data for each new lead.

---

## What You're Building

A **daily cron job** running on OpenClaw (hosted on Railway) that:
1. Queries HubSpot for leads created X days ago
2. For each lead, pulls engagement data from HubSpot, session data from Fullstory, and call transcripts from Gong
3. Synthesizes a per-lead analysis and delivers the summary to you via Slack

---

## Prerequisites

Gather these before you start:

- [ ] **Railway account** (paid plan — free tier has only 1 GB memory, and OpenClaw needs 2+ GB)
- [ ] **Anthropic API key** — from console.anthropic.com (funded account)
- [ ] **Slack workspace with admin access** — for creating a Slack app and receiving analysis results
- [ ] **HubSpot Private App Access Token** — Settings → Integrations → Private Apps → create with CRM read scopes
- [ ] **Fullstory API Key** — Settings → Integrations → API Keys
- [ ] **Gong API credentials** — Company Settings → API → create key (Access Key + Secret)

---

## Step 1 · Deploy on Railway

1. Go to the official deploy link:
   **https://railway.com/deploy/clawdbot-railway-template**

2. Click **Deploy** — Railway creates the service from the OpenClaw template.

3. While it's deploying, configure three things in Railway dashboard:

### 1a. Volume

The template pre-configures a volume at **`/data`** — verify it exists under **Settings** → **Volumes**. If missing, add one mounted at `/data`. This is required — without it, your config and cron jobs are lost on every redeploy.

### 1b. Set Variables
The template comes with 3 pre-configured environment variables plus a setup password field. Fill them in:

**Setup password** (top of the form):
- Choose a strong password — this protects your `/setup` wizard from the public internet

**Pre-configured variables** (already populated with sensible defaults):

| Variable | Default Value | What it does |
|----------|--------------|--------------|
| `CLAWDBOT_STATE_DIR` | `/data/.clawdbot` | Where configs, auth, and sessions live |
| `CLAWDBOT_GATEWAY_TOKEN` | `${{secret(64, ...)}}` | Auto-generated token that protects your gateway (leave as-is) |
| `CLAWDBOT_WORKSPACE_DIR` | `/data/workspace` | Where agent files and memory reside |

**Add this variable manually** (not pre-configured by the template):

| Variable | Value | What it does |
|----------|-------|--------------|
| `NODE_OPTIONS` | `--max-old-space-size=768` | Prevents JavaScript heap out-of-memory crashes during setup |

> **Note:** This template uses the older `CLAWDBOT_*` naming (from before the project was renamed to OpenClaw). This is normal — the aliases still work. Don't change them to `OPENCLAW_*` unless the template docs say to.

### 1c. Enable Public Networking
- Go to **Settings** → **Networking**
- The template auto-configures public networking on port **`8080`** — verify it's enabled and note your public domain (something like `your-app.up.railway.app`)

> ⚠️ **Port 8080, not 3000.** The template docs may mention port 3000, but the container actually listens on 8080. You can confirm this in the Railway logs — look for `[wrapper] listening on :8080`.

---

## Step 2 · Create the Slack App

Do this **before** running the setup wizard, so you have both tokens ready.

### 2a. Create the app
- Go to **https://api.slack.com/apps**
- Click **"Create New App"** → **"From scratch"**
- Name it (e.g., "OpenClaw Lead Analyzer") and select your workspace
- Click **Create App**

### 2b. Enable Socket Mode
- In the sidebar, go to **Socket Mode**
- Toggle **Enable Socket Mode** to ON
- This must be done **before** configuring Event Subscriptions (it changes how events are delivered and removes the Request URL requirement)

### 2c. Generate the App Token (`xapp-...`)
- In the sidebar, go to **Basic Information**
- Scroll to **App-Level Tokens**
- Click **"Generate Token and Scopes"**
- Name it (e.g., "openclaw-socket") and add scope: **`connections:write`**
- Click **Generate**
- Copy the token (starts with `xapp-`) — save it somewhere, you'll need it in Step 3

### 2d. Add Bot Token Scopes
- In the sidebar, go to **OAuth & Permissions**
- Scroll to **Bot Token Scopes** and add these scopes:

| Scope | Purpose |
|-------|---------|
| `chat:write` | Send messages |
| `channels:read` | List channels |
| `channels:history` | Read channel messages |
| `groups:history` | Read private channel messages |
| `im:history` | Read DMs |
| `mpim:history` | Read group DMs |
| `users:read` | Look up user info |
| `app_mentions:read` | Respond to @mentions |
| `reactions:read` | Read reactions |
| `reactions:write` | Add reactions (ack emoji) |
| `pins:read` | Read pins |
| `pins:write` | Manage pins |
| `emoji:read` | Custom emoji |
| `files:read` | Read files |
| `files:write` | Upload files |

- Scroll to the top and click **"Install to Workspace"** → **Authorize**
- Copy the **Bot User OAuth Token** (starts with `xoxb-`) — save it

### 2e. Enable Event Subscriptions

> **Important:** With Socket Mode ON, this page should **not** show a Request URL field. If it does, go back to Socket Mode and confirm the toggle is ON.

- In the sidebar, go to **Event Subscriptions**
- Toggle **Enable Events** to ON
- Under **"Subscribe to bot events"**, add all of these:

| Event | What it does |
|-------|-------------|
| `message.im` | DMs to the bot |
| `message.channels` | Messages in public channels |
| `message.groups` | Messages in private channels |
| `message.mpim` | Group DM messages |
| `app_mention` | When someone @mentions the bot |
| `reaction_added` | Reaction added to messages |
| `reaction_removed` | Reaction removed |
| `member_joined_channel` | User joins a channel |
| `member_left_channel` | User leaves a channel |
| `channel_rename` | Channel renamed |
| `pin_added` | Message pinned |
| `pin_removed` | Message unpinned |

- Click **Save Changes**

### 2f. Enable Messages Tab (required for DMs)
- In the sidebar, go to **App Home**
- Scroll to **"Show Tabs"**
- Check **"Messages Tab"**
- Check **"Allow users to send Slash commands and messages from the messages tab"**
- Save

### 2g. Reinstall the App
After adding events and scopes, you **must** reinstall:
- Go to **OAuth & Permissions** → **Reinstall to Workspace** → **Allow**
- Your `xoxb-` token stays the same

### 2h. Restart Slack Client

> ⚠️ **Critical:** After all Slack app config changes, fully quit and reopen Slack (Cmd+Q on Mac, not just close the window). The "Sending messages to this app has been turned off" notice won't clear until the Slack client fully restarts.

---

## Step 3 · Run the Setup Wizard

1. Open your browser and go to:
   ```
   https://<your-railway-domain>/setup
   ```

2. Enter your `SETUP_PASSWORD`

3. **Choose model/auth provider:**
   - Provider group: **Anthropic - Claude Code CLI + API key**
   - Auth method: Change from "Anthropic token (paste setup-token)" to **"API key"** — the setup-token option expects an OAuth token (`sk-ant-oat01-...`), not a standard API key
   - Paste your **Anthropic API key** (`sk-ant-api03-...`)
   - The wizard will let you pick a model — choose **claude-sonnet-4-5** (not opus — it's 5x cheaper and more than capable)

4. **Add Slack tokens:**
   - Paste the `xoxb-...` token into **Slack bot token**
   - Paste the `xapp-...` token into **Slack app token**

5. Click **Run setup**

6. After setup completes, your setup page stays at:
   ```
   https://<your-railway-domain>/setup
   ```

---

## Step 4 · Enable the Slack Plugin

> ⚠️ **This is the step most people miss.** The setup wizard configures Slack tokens in the config, but the Slack plugin is **disabled by default** in the Railway template. Without enabling it, the gateway starts but Slack never connects.

### How to tell if Slack is not loading
Check Railway logs. You'll see `[canvas]`, `[heartbeat]`, `[gateway]`, `[browser/service]` — but **no `[slack]` lines at all**. The Channels table in `openclaw status` will be empty.

### Fix it

1. Go to the `/setup` page → scroll to **Debug console**
2. From the command dropdown, select **`openclaw plugins list`** → click **Run**
3. Confirm `@openclaw/slack` shows as `disabled`
4. From the dropdown, select **`openclaw plugins enable <name>`**
5. In the **Optional arg** field, type: `slack`
6. Click **Run** — you should see: `Enabled plugin "slack". Restart the gateway to apply.`
7. From the dropdown, select **`gateway.restart (wrapper-managed)`** → click **Run**

### Verify it worked
Check Railway logs. You should now see:
```
[slack] [default] starting provider
[slack] socket mode connected
```

And `openclaw status` (from the debug console) should show Slack in the Channels table.

---

## Step 5 · Configure Slack Access Policy

After Slack connects, configure how the bot handles DMs and channels. Go to `/setup` → **Config editor** (advanced section) and set the Slack channel config:

```json
{
  "channels": {
    "slack": {
      "enabled": true,
      "mode": "socket",
      "botToken": "xoxb-...",
      "appToken": "xapp-...",
      "groupPolicy": "open",
      "dm": {
        "enabled": true,
        "policy": "open",
        "allowFrom": ["*"]
      }
    }
  }
}
```

> **Schema version note (v2026.2.9):** This template uses an older config schema. Use nested `dm.policy` and `dm.allowFrom` — **not** the top-level `dmPolicy` and `allowFrom` shown in the current OpenClaw docs. The docs describe the newer schema (v2026.2.19+). If you get "Unrecognized key" errors, this is why.

### DM Policy Options

| Policy | Behavior |
|--------|----------|
| `pairing` (default) | User must send a pairing code, you approve it via debug console |
| `open` | Anyone can DM the bot (use `allowFrom: ["*"]` or specific Slack user IDs) |
| `allowlist` | Only users in `allowFrom` list can DM |
| `disabled` | DMs turned off |

For a private workspace where only you use it, `open` with `["*"]` is fine. For shared workspaces, use `allowlist` with specific user IDs.

---

## Step 6 · Verify It Works

**Test via DM:**
1. Open Slack and find your bot under **Apps** (or search for it)
2. Send a DM: **"Hello, are you working?"**
3. The bot should respond within a few seconds

**Test via channel:**
1. Invite your bot to a channel: `/invite @OpenClaw Lead Analyzer`
2. Mention the bot: **"@OpenClaw Hello"**

If the bot doesn't respond, check:
- Railway logs for errors
- `openclaw status` from the debug console — Channels table should show Slack as `enabled` / `connected`
- `openclaw doctor` from the debug console for diagnostics

---

## Step 7 · Enable Cron

From the debug console (or Railway shell if you have access):

Use `openclaw config get <path>` from the dropdown:
- Check: `cron.enabled` — if not `true`, you'll need to set it via the config editor

In the config editor, add or merge:

```json
{
  "cron": {
    "enabled": true,
    "maxConcurrentRuns": 2,
    "sessionRetention": "24h"
  }
}
```

Save and restart the gateway.

---

## Step 8 · Add MCP Servers

In the config editor at `/setup`, add the MCP servers under the `agents` section (merge with existing config — don't overwrite everything):

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "mcp": {
          "servers": [
            {
              "name": "hubspot",
              "command": "npx",
              "args": ["-y", "@hubspot/mcp-server"],
              "env": {
                "PRIVATE_APP_ACCESS_TOKEN": "pat-na1-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
              }
            },
            {
              "name": "fullstory",
              "command": "npx",
              "args": ["-y", "@fullstory/mcp-server"],
              "env": {
                "FULLSTORY_API_KEY": "your-fullstory-api-key"
              }
            },
            {
              "name": "gong",
              "command": "npx",
              "args": ["-y", "@gong-io/mcp-server"],
              "env": {
                "GONG_ACCESS_KEY": "your-gong-access-key",
                "GONG_ACCESS_KEY_SECRET": "your-gong-secret"
              }
            }
          ]
        }
      }
    ]
  }
}
```

> **Note on MCP packages:** The HubSpot MCP server (`@hubspot/mcp-server`) is an official npm package. Fullstory and Gong may not have official npm MCP packages yet. If `npx -y @fullstory/mcp-server` or `@gong-io/mcp-server` fails, search npmjs.com or GitHub for community MCP servers, or use the `openclaw-mcp-plugin` for HTTP/SSE transport if the service has a remote MCP endpoint.

After saving, restart the gateway from the debug console.

---

## Step 9 · Test Each MCP

Message your bot in Slack:

> "Use HubSpot to find any contact created in the last 3 days. Show me their name and email."

> "Use Fullstory to list recent sessions."

> "Use Gong to list calls from the last 7 days."

If any fail, check logs via `openclaw logs --tail N` from the debug console (use optional arg: `50`).

---

## Step 10 · Create the Cron Job

Message your bot in Slack with the cron creation command, or use the config editor to add a cron entry. From Railway shell (if available):

```bash
openclaw cron add \
  --name "Lead triangulation" \
  --cron "0 8 * * 1-5" \
  --tz "Asia/Jerusalem" \
  --session isolated \
  --announce \
  --message "Run the daily lead triangulation analysis.

STEP 1 — FIND LEADS:
Use HubSpot to search for contacts where createdate is between 8 days ago and 7 days ago (contacts created exactly 7 days ago). Include properties: email, firstname, lastname, company, lifecyclestage, hs_lead_status, hubspot_owner_id.

If no leads match, respond: No leads reached the 7-day mark today. Stop here.

STEP 2 — FOR EACH LEAD, GATHER DATA:

a) HubSpot deep-dive:
   - Get all engagement activity (emails sent/received, meetings booked, calls logged, tasks, notes)
   - Get deal associations and current pipeline stage
   - Get form submissions and page views if available
   - Note the assigned owner

b) Fullstory analysis:
   - Search sessions by the lead's email address
   - Get session context: navigation patterns, pages visited, rage clicks, dead clicks, error clicks, mouse thrash, time on key pages
   - Identify which product areas they explored and any UX friction points

c) Gong call analysis:
   - Search calls by the lead's email address
   - For any calls found, get the transcript
   - Extract: pain points mentioned, objections raised, competitors discussed, buying signals, budget indicators, next steps agreed

STEP 3 — SYNTHESIZE PER LEAD:
For each lead, produce:
- Lead: [name] ([email]) — Owner: [owner name]
- Engagement Score (Low/Medium/High): based on HubSpot activity volume and recency
- Product Interest: features or areas from Fullstory navigation + Gong topics
- Digital Experience Quality: Fullstory frustration signals, broken flows, confusion
- Sales Conversation Quality: Gong — objections handled, next steps established
- Risk Flags: low engagement, frustrated sessions, unresolved objections, competitor mentions, no follow-up
- Recommended Next Action: one specific suggestion for the sales rep

STEP 4 — DELIVER:
Compile all analyses into one summary with date and lead count header."
```

### What each flag does:

| Flag | Purpose |
|------|---------|
| `--cron "0 8 * * 1-5"` | 8:00 AM, Monday–Friday |
| `--tz "Asia/Jerusalem"` | Israel Standard Time |
| `--session isolated` | Clean session per run, doesn't clutter your chat |
| `--announce` | Delivers the summary to your Slack channel |

---

## Step 11 · Test It

Force-run immediately (from Railway shell or by messaging the bot):

```bash
openclaw cron run lead-triangulation --force
```

Watch logs from the debug console: `openclaw logs --tail N` (arg: `100`).

You should receive the analysis summary in Slack within 2–5 minutes (depending on number of leads and Gong transcripts).

---

## Step 12 · Verify the Schedule

```bash
openclaw cron list --verbose
```

Confirm:
- `lead-triangulation` — enabled: `true`
- Next run: tomorrow at 8:00 AM IST (or next weekday)
- Schedule: `0 8 * * 1-5`

---

## Ongoing Management

### Change the lead age (e.g., 14 days instead of 7)

```bash
openclaw cron edit lead-triangulation --message "(updated prompt with 14/15 instead of 7/8)"
```

### Use a different model for the cron job

```bash
openclaw cron edit lead-triangulation --model "anthropic/claude-haiku-4-5"
```

### Pause without deleting

```bash
openclaw cron disable lead-triangulation
openclaw cron enable lead-triangulation    # re-enable
```

### Backup your setup

Download everything (config, credentials, workspace, cron definitions):

```
https://<your-railway-domain>/setup/export
```

Save this periodically — it lets you migrate to another host without losing anything.

### Update OpenClaw

From the debug console or Railway shell:

```bash
openclaw update
```

Then verify with `openclaw doctor`.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| **Slack channel not in `openclaw status`** | The Slack plugin is disabled. Run `openclaw plugins enable slack` from the debug console, then restart the gateway. This is the #1 issue people hit. |
| **No `[slack]` lines in logs at all** | Same as above — plugin not enabled. |
| **`[slack] socket mode connected` but bot doesn't respond** | Check `dm.policy` in config. If set to `pairing`, you need to approve the pairing code. Set to `open` for testing. Also check Event Subscriptions in Slack app — `message.im` must be subscribed. |
| **"Unrecognized key" config errors** | Template v2026.2.9 uses legacy schema. Use `dm.policy` (nested), not `dmPolicy` (top-level). See Step 5 note. |
| **"JavaScript heap out of memory"** | Add `NODE_OPTIONS=--max-old-space-size=768` to Railway variables. Or upgrade Railway plan to 2+ GB. |
| **"Application failed to respond"** | Port mismatch. Ensure Railway Networking is set to port `8080`, not 3000. |
| **Bot doesn't respond in channels** | Bot must be invited to the channel. Messages must @mention the bot (default behavior). Check `groupPolicy` in config. |
| **Config changes not applying** | Restart gateway from debug console: `gateway.restart (wrapper-managed)`. |
| **Slack shows "Sending messages turned off"** | Fully quit and reopen Slack (Cmd+Q on Mac). |
| **Cron never fires** | Verify `cron.enabled: true` in config. Check gateway is running. Railway paid plan required. |
| **MCP server fails to start** | Check logs for specific error. Verify API tokens are correct. |
| **Config lost after redeploy** | Volume not mounted at `/data`. Re-add it in Railway settings. |
| **Gateway won't start** | Check Railway Logs tab. Run `openclaw doctor` from debug console. |

---

## Debug Console Commands

All available from `/setup` → Debug console dropdown:

| Command | What it does |
|---------|-------------|
| `openclaw --version` | Show installed version |
| `openclaw status` | Full status including channels, sessions, security audit |
| `openclaw health` | Quick health check |
| `openclaw doctor` | Diagnostic report with fix suggestions |
| `openclaw logs --tail N` | Recent log lines (put number in optional arg) |
| `openclaw config get <path>` | Read config value (e.g., `channels.slack`) |
| `openclaw plugins list` | Show all plugins and their enabled/disabled state |
| `openclaw plugins enable <name>` | Enable a plugin (e.g., `slack`) |
| `openclaw devices list` | List paired devices |
| `openclaw devices approve <requestId>` | Approve a pairing request |
| `gateway.restart` | Restart gateway (wrapper-managed) |
| `gateway.stop` | Stop gateway |
| `gateway.start` | Start gateway |

---

## Cost Optimization

The template defaults to `claude-opus-4-6` which is the most expensive model ($15/$75 per million tokens input/output). For lead triangulation:

| Model | Input/Output Cost | Estimated daily cost (5 leads) |
|-------|-------------------|-------------------------------|
| `claude-opus-4-6` | $15 / $75 | $0.50–1.50/day |
| `claude-sonnet-4-5` | $3 / $15 | $0.10–0.30/day |
| `claude-haiku-4-5` | $0.80 / $4 | $0.03–0.10/day |

**Recommendation:** Use `claude-sonnet-4-5` for the main agent model. It handles the triangulation analysis well at 5x lower cost. Change it in the config editor under the model field, then restart the gateway.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   Railway                        │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │       OpenClaw Container (always-on)       │ │
│  │                                            │ │
│  │  ┌──────────┐  ┌───────────────────────┐  │ │
│  │  │ Cron     │  │ Agent (Claude Sonnet) │  │ │
│  │  │ Scheduler│──│                       │  │ │
│  │  │ 8AM M-F  │  │ ┌───────┐ ┌────────┐ │  │ │
│  │  │ IST      │  │ │HubSpot│ │Fulstory│ │  │ │
│  │  │          │  │ │  MCP  │ │  MCP   │ │  │ │
│  │  └──────────┘  │ └───────┘ └────────┘ │  │ │
│  │                │ ┌───────┐            │  │ │
│  │                │ │ Gong  │            │  │ │
│  │                │ │  MCP  │            │  │ │
│  │                │ └───────┘            │  │ │
│  │                └───────────┬──────────┘  │ │
│  │                            │ announce    │ │
│  │  ┌─────────┐   ┌──────────▼──────────┐  │ │
│  │  │ /data   │   │ Slack (Socket Mode) │  │ │
│  │  │ Volume  │   └─────────────────────┘  │ │
│  │  │(persist)│                             │ │
│  │  └─────────┘                             │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
│  Browser access:                                 │
│  /setup    — setup wizard + debug console        │
│  /openclaw — Control UI (config, logs, cron)     │
└─────────────────────────────────────────────────┘
                         │
                    Socket Mode
                         │
                         ▼
                   💬 Slack
                   DMs + #lead-analysis channel
```

---

## Version & Compatibility Notes

| Item | Value |
|------|-------|
| Template | `vignesh07/clawdbot-railway-template` |
| OpenClaw version | `v2026.2.9` |
| Latest available | `v2026.2.23` (via `openclaw update`) |
| Config naming | Uses legacy `CLAWDBOT_*` env vars |
| Config schema | Legacy nested `dm.policy` / `dm.allowFrom` (not top-level `dmPolicy`) |
| Container port | `8080` |
| Min memory | 2 GB (1 GB may work with `NODE_OPTIONS` but is tight) |
