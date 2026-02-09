# OpenClaw Mission Control — Agent Setup Guide

You are setting up the OpenClaw Mission Control dashboard for the user. Follow these instructions exactly. Do not skip steps. Do not improvise alternatives. Every step has been tested.

This guide works with any AI agent — Claude, GPT, Gemini, LLaMA, or any other model.

---

## What This Is

A real-time web dashboard for monitoring AI agent swarms. It reads task JSON files from disk, displays them on a Kanban board with live agent status, token usage tracking, and activity feeds. It can optionally connect to PostgreSQL for richer data via a bridge script.

## Prerequisites

Before starting, confirm these exist on the machine:
- **Node.js 18+** (`node --version`)
- **npm** (`npm --version`)
- **git** (`git --version`)
- **PostgreSQL** (optional — only needed for the DB bridge. Docker works: `docker ps | grep postgres`)

If any prerequisite is missing, install it before continuing.

---

## Installation

### Step 1: Clone and install

```bash
git clone https://github.com/bokiko/openClaw-dashboard.git
cd openClaw-dashboard
npm install
```

### Step 2: Create environment file

```bash
cp .env.example .env.local
```

### Step 3: Configure .env.local

Edit `.env.local` with the correct values:

```bash
# REQUIRED: Path to the directory containing task JSON files.
# The dashboard reads *.json from this directory every 30 seconds.
# Use an absolute path in production.
OPENCLAW_TASKS_DIR=./tasks

# RECOMMENDED: Protect the API in production. Set a random string.
# If left empty, the API is open (fine for localhost only).
OPENCLAW_API_KEY=
NEXT_PUBLIC_OPENCLAW_API_KEY=

# OPTIONAL: PostgreSQL connection for the bridge script.
# Only needed if you want to sync sessions/handoffs/token usage from a DB.
DATABASE_URL=
```

### Step 4: Create the tasks directory

```bash
mkdir -p tasks
```

### Step 5: Personalize the dashboard

**IMPORTANT: Before starting the server, run the personalization step below.** This creates a `settings.json` that persists across updates. See the [Personalization](#personalization) section.

### Step 6: Start the dashboard

For development:
```bash
npm run dev
```

For production:
```bash
npm run build
npm start
```

The dashboard runs on `http://localhost:3000` by default.

---

## Personalization

The dashboard is fully personalizable via a `settings.json` file in the project root. This file is gitignored, so it survives `git pull` updates — the user's preferences are never overwritten.

### How it works

1. Copy the example: `cp settings.example.json settings.json`
2. Edit `settings.json` with the user's preferences
3. The dashboard reads `settings.json` on every request (cached for 5 seconds)
4. Changes take effect on the next page load — no restart needed

### Interactive setup (ask the user)

**You MUST ask the user these questions before writing `settings.json`.** Present each question clearly, show the available options, and use their answers to build the file. If the user says "just use defaults" or skips a question, use the default value.

#### Question 1: Dashboard name
> "What would you like to name your Mission Control dashboard?"
>
> Default: `"OpenClaw"`
> Examples: `"Alpha Squad"`, `"Project Phoenix"`, `"Neural Ops"`

Maps to: `name` field

#### Question 2: Subtitle
> "What subtitle should appear below the name?"
>
> Default: `"Mission Control"`
> Examples: `"Command Center"`, `"Agent HQ"`, `"Control Room"`

Maps to: `subtitle` field

#### Question 3: Theme
> "Which theme do you prefer?"
>
> Options:
> - `dark` (default) — Dark background, light text. Best for low-light environments.
> - `light` — Light background, dark text. Best for bright environments.

Maps to: `theme` field

#### Question 4: Accent color
> "Pick an accent color for the dashboard:"
>
> Options:
> - `green` (default) — Fresh and balanced
> - `blue` — Professional and calm
> - `purple` — Bold and creative
> - `orange` — Energetic and warm
> - `red` — Intense and urgent
> - `cyan` — Cool and technical
> - `amber` — Warm and golden
> - `pink` — Vibrant and modern

Maps to: `accentColor` field

#### Question 5: Logo icon
> "Which icon should appear in the header?"
>
> Options:
> - `zap` (default) — Lightning bolt
> - `brain` — Brain
> - `bot` — Robot
> - `flame` — Fire
> - `shield` — Shield
> - `rocket` — Rocket
> - `sparkles` — Sparkles
> - `cpu` — Processor chip
> - `eye` — Eye
> - `activity` — Heartbeat pulse

Maps to: `logoIcon` field

#### Question 6: Repository URL (optional)
> "Would you like to link to a GitHub/GitLab repository? If so, paste the URL. Otherwise say 'skip'."
>
> Default: `null` (no link shown)

Maps to: `repoUrl` field (set to `null` if skipped)

#### Question 7: Card density
> "How dense should task cards be?"
>
> Options:
> - `comfortable` (default) — More spacing, easier to read
> - `compact` — Tighter layout, fits more tasks on screen

Maps to: `cardDensity` field

#### Question 8: Panels
> "Which metric panels should be visible?"
>
> Options (multi-select):
> - Metrics Panel — task throughput, status distribution charts (default: on)
> - Token Usage Panel — token consumption charts and model breakdown (default: on)

Maps to: `showMetricsPanel` and `showTokenPanel` fields

#### Question 9: Time display
> "How should timestamps be displayed?"
>
> Options:
> - `utc` (default) — All times in UTC
> - `local` — Times in the user's local timezone

Maps to: `timeDisplay` field

### Writing settings.json

After collecting answers, write `settings.json` to the project root. Here is the full schema with all defaults:

```json
{
  "name": "OpenClaw",
  "subtitle": "Mission Control",
  "repoUrl": null,
  "logoIcon": "zap",
  "theme": "dark",
  "accentColor": "green",
  "backgroundGradient": {
    "topLeft": "rgba(70,167,88,0.05)",
    "bottomRight": "rgba(62,99,221,0.05)"
  },
  "cardDensity": "comfortable",
  "showMetricsPanel": true,
  "showTokenPanel": true,
  "refreshInterval": 30000,
  "timeDisplay": "utc",
  "agents": null
}
```

**Background gradient tip:** Match the `topLeft` color to the accent color at 5% opacity. Here are the gradient presets per accent:

| Accent | topLeft | bottomRight |
|--------|---------|-------------|
| green | `rgba(70,167,88,0.05)` | `rgba(62,99,221,0.05)` |
| blue | `rgba(62,99,221,0.05)` | `rgba(142,78,198,0.05)` |
| purple | `rgba(142,78,198,0.05)` | `rgba(62,99,221,0.05)` |
| orange | `rgba(247,107,21,0.05)` | `rgba(255,178,36,0.05)` |
| red | `rgba(229,77,46,0.05)` | `rgba(247,107,21,0.05)` |
| cyan | `rgba(0,162,199,0.05)` | `rgba(62,99,221,0.05)` |
| amber | `rgba(255,178,36,0.05)` | `rgba(247,107,21,0.05)` |
| pink | `rgba(232,121,164,0.05)` | `rgba(142,78,198,0.05)` |

### Custom agent roster (optional)

If the user has specific agents they want displayed on the agent strip, ask them. Otherwise set `agents` to `null` to use the default roster.

Custom format:
```json
{
  "agents": [
    { "id": "alpha", "name": "Alpha", "letter": "A", "color": "#3e63dd", "role": "Lead Engineer", "badge": "lead" },
    { "id": "beta", "name": "Beta", "letter": "B", "color": "#46a758", "role": "Code & Writing", "badge": "spc" }
  ]
}
```

- `id`: unique lowercase identifier
- `name`: display name
- `letter`: single character shown in the avatar circle
- `color`: hex color for the agent's avatar (must be `#` + 6 hex digits)
- `role`: short description
- `badge`: optional, `"lead"` or `"spc"`

### Settings survive updates

The `settings.json` file is listed in `.gitignore`. When the user runs `git pull` or clicks the update button, their settings are preserved. They never need to re-personalize after an update.

---

## Production Deployment

### Use a process manager

The dashboard's update button (`/api/update`) pulls code and restarts the process. This requires a process manager:

```bash
# PM2 (recommended)
npm install -g pm2
pm2 start npm --name "mission-control" -- start
pm2 save
pm2 startup

# Or systemd (create /etc/systemd/system/mission-control.service)
# Or Docker with restart: always
```

### Bind to all interfaces (LAN access)

Next.js binds to `0.0.0.0` by default in production. For dev mode:
```bash
npm run dev -- -H 0.0.0.0
```

### Set API key for production

Never leave `OPENCLAW_API_KEY` empty in production. Generate a key:
```bash
echo "OPENCLAW_API_KEY=$(openssl rand -hex 32)" >> .env.local
echo "NEXT_PUBLIC_OPENCLAW_API_KEY=$(grep OPENCLAW_API_KEY .env.local | head -1 | cut -d= -f2)" >> .env.local
```

---

## PostgreSQL Bridge Setup (Optional)

If you have a PostgreSQL database with session/handoff data, the bridge script syncs that data into task JSON files.

### Step 1: Set DATABASE_URL

```bash
# In .env.local
DATABASE_URL=postgresql://user:password@localhost:5432/your_database
```

### Step 2: Run database migrations

```bash
npm run setup-db
```

### Step 3: Run the sync

One-time sync:
```bash
npm run sync
```

Watch mode (re-syncs on file changes):
```bash
npm run sync:watch
```

For continuous syncing, add a cron job:
```bash
# Sync every 30 seconds (matches dashboard refresh interval)
* * * * * cd /path/to/openClaw-dashboard && npm run sync >> /tmp/openclaw-sync.log 2>&1
* * * * * sleep 30 && cd /path/to/openClaw-dashboard && npm run sync >> /tmp/openclaw-sync.log 2>&1
```

---

## Token Usage Tracking

Token usage appears automatically when task JSON files contain a `usage` array. Three ways to populate it:

### Option A: dispatch-task.sh (automatic)

Wraps any agent command and auto-captures token usage:

```bash
npm run dispatch -- my-task-id "Fix the authentication bug"
npm run dispatch -- refactor-api "Refactor the API layer" --model claude-sonnet-4-5
```

### Option B: log-usage (manual/scripted)

Call after any agent run from your own automation:

```bash
npm run log-usage -- \
  --session-id my-session \
  --task-id my-task \
  --input 15000 \
  --output 4200 \
  --model claude-opus-4
```

### Option C: sync-db bridge (from database)

If token usage is logged to the `token_usage` PostgreSQL table by other tools, `npm run sync` picks it up automatically.

---

## Task JSON File Format

Drop JSON files into `OPENCLAW_TASKS_DIR`. The dashboard is flexible with field names:

```json
{
  "id": "task-001",
  "title": "Implement auth middleware",
  "description": "Add JWT validation to all API routes",
  "status": "in-progress",
  "priority": "high",
  "claimed_by": "spark",
  "tags": ["backend", "security"],
  "created_at": "2025-01-15T10:00:00Z",
  "usage": [
    {
      "inputTokens": 12100,
      "outputTokens": 6300,
      "model": "claude-opus-4",
      "provider": "anthropic",
      "timestamp": 1705312800000
    }
  ]
}
```

### Status values

| Your value | Dashboard shows |
|------------|-----------------|
| `complete`, `completed`, `done`, `approved` | Done |
| `in-progress`, `in_progress`, `active`, `working` | In Progress |
| `review`, `submitted`, `pending_review` | Review |
| `assigned`, `claimed` | Assigned |
| `waiting`, `blocked`, `paused` | Waiting |
| anything else | Inbox |

### Priority values

| Your value | Dashboard shows |
|------------|-----------------|
| `urgent`, `p0`, `critical` | Urgent (red) |
| `high`, `p1` | High (amber) |
| anything else | Normal |

### Assignee detection

Checks in order: `claimed_by` -> `assignee` -> first `deliverables[].assignee`

### Usage array

Each entry in `usage` represents one API call. Fields:
- `inputTokens` (required) — also accepts `input_tokens`
- `outputTokens` (required) — also accepts `output_tokens`
- `cacheReadTokens` (optional) — also accepts `cache_read_tokens`
- `cacheWriteTokens` (optional) — also accepts `cache_write_tokens`
- `model` (optional) — e.g. "claude-opus-4", "gpt-4o", "gemini-pro"
- `provider` (optional) — "anthropic", "openai", "google", etc.
- `timestamp` (optional) — epoch milliseconds

---

## Architecture

```
Data flow:

  [PostgreSQL] --sync-db.ts--> [tasks/*.json] --data.ts--> [/api/data] --useSwarmData--> [UI]
                                     ^
  [dispatch-task.sh] --log-usage.ts--+
  [your scripts] --log-usage.ts------+
```

Key files:
- `src/lib/settings.ts` — Settings loader (reads settings.json, caches 5s)
- `src/lib/data.ts` — Server-side: reads task JSON files, computes stats and token aggregates
- `src/app/api/data/route.ts` — API endpoint with auth + rate limiting
- `src/app/api/update/route.ts` — Dashboard self-update endpoint (git pull + restart)
- `src/lib/useSwarmData.ts` — Client hook, polls /api/data every 30s
- `src/components/TokenMetricsPanel.tsx` — Token usage charts (trend + model breakdown)
- `scripts/sync-db.ts` — PostgreSQL -> task JSON bridge
- `scripts/log-usage.ts` — Token usage writer (DB + file)
- `scripts/dispatch-task.sh` — Agent wrapper with auto-capture
- `scripts/ensure-tables.ts` — Database migration runner

---

## npm Scripts Reference

| Command | What it does |
|---------|-------------|
| `npm run dev` | Start dev server with hot reload |
| `npm run build` | Production build |
| `npm start` | Start production server |
| `npm run sync` | Sync PostgreSQL -> task JSON files (one-time) |
| `npm run sync:watch` | Sync with file watch (re-runs on changes) |
| `npm run setup-db` | Run database migrations (creates token_usage table) |
| `npm run log-usage -- [args]` | Log token usage to DB + file |
| `npm run dispatch -- [task-id] [prompt]` | Run agent with auto-capture |
| `npm run lint` | Run ESLint |

---

## Verification Checklist

After setup, verify everything works:

1. `npx tsc --noEmit` — TypeScript compiles with zero errors
2. `npm run dev` — Dashboard loads at http://localhost:3000
3. Dashboard shows the custom name and accent color from settings.json
4. Theme matches what the user chose (dark or light)
5. Token Usage panel shows "No usage data yet" (normal if no data)
6. Click a task card -> modal appears centered, scrollable
7. Update button shows "Already up to date" (if on latest commit)

If using PostgreSQL bridge:
8. `npm run setup-db` — prints "All migrations applied"
9. `npm run sync` — prints summary with session/handoff/token counts
10. Dashboard populates with synced data after refresh

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `ENOENT: tasks directory not found` | Create it: `mkdir -p tasks` |
| Dashboard shows empty state | Add task JSON files to `OPENCLAW_TASKS_DIR` or run `npm run sync` |
| Token panel always empty | Tasks need a `usage` array — use `npm run dispatch` or `npm run log-usage` |
| `npm run sync` fails with connection error | Check `DATABASE_URL` in `.env.local`, verify PostgreSQL is running |
| `npm run setup-db` fails | Ensure PostgreSQL is accessible and the user has CREATE TABLE permission |
| Update button shows error | Ensure the git working tree is clean (`git status`) and the remote is reachable |
| Settings not applying | Check `settings.json` exists in project root and is valid JSON |
| Theme/colors look wrong | Verify `accentColor` is one of: green, blue, purple, orange, red, cyan, amber, pink |
| `next build` fails on `/_global-error` | Known Next.js 16 bug — does not affect functionality, ignore it |

---

## Security Notes

- API key auth protects `/api/data` and `/api/update` in production
- Rate limiting: 60 requests/min per IP on `/api/data`
- Path traversal protection on task file loading
- File size limit: 1MB per task JSON
- The `/api/update` endpoint runs `git pull --ff-only` (no force pulls, no rebase)
- Error messages are sanitized (no stack traces exposed)
- Agent colors are validated (must be `#` + 6 hex digits)

---

## Do Not

- Do not edit `postcss.config.js` — it must stay as CommonJS (`.js`, not `.mjs`)
- Do not add `.env` files to git — they are gitignored (`.env.example` is the exception)
- Do not put non-JSON files in `OPENCLAW_TASKS_DIR` — only `*.json` is read
- Do not use `--force` with git operations from the update endpoint
- Do not commit `tasks/` contents — they are generated/ephemeral and gitignored
- Do not commit `settings.json` — it is user-specific and gitignored
