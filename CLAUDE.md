# OpenClaw Mission Control — Agent Setup Guide

You are setting up the OpenClaw Mission Control dashboard. Follow these instructions exactly. Do not skip steps. Do not improvise alternatives. Every step has been tested.

## What This Is

A real-time web dashboard for monitoring AI agent swarms. It reads task JSON files from disk, displays them on a Kanban board with live agent status, token usage tracking, and activity feeds. It can optionally connect to PostgreSQL for richer data via a bridge script.

## Prerequisites

Before starting, confirm these exist on the machine:
- **Node.js 18+** (`node --version`)
- **npm** (`npm --version`)
- **git** (`git --version`)
- **PostgreSQL** (optional — only needed for the DB bridge. Docker works: `docker ps | grep postgres`)

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

Edit `.env.local` with the correct values. Here is what each variable does:

```bash
# REQUIRED: Path to the directory containing task JSON files.
# The dashboard reads *.json from this directory every 30 seconds.
# Use an absolute path in production.
OPENCLAW_TASKS_DIR=./tasks

# RECOMMENDED: Protect the API in production. Set a random string.
# If left empty, the API is open (fine for localhost only).
OPENCLAW_API_KEY=
NEXT_PUBLIC_OPENCLAW_API_KEY=

# OPTIONAL: Custom name shown in the header (defaults to "OpenClaw")
NEXT_PUBLIC_DASHBOARD_NAME=

# OPTIONAL: GitHub/GitLab repo URL — shows a link icon in the header
NEXT_PUBLIC_REPO_URL=

# OPTIONAL: PostgreSQL connection for the bridge script.
# Only needed if you want to sync sessions/handoffs/token usage from a DB.
# Format: postgresql://user:password@host:port/database
DATABASE_URL=
```

### Step 4: Create the tasks directory

```bash
mkdir -p tasks
```

### Step 5: Start the dashboard

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

If you have a PostgreSQL database with session/handoff data (e.g., from Continuous Claude), the bridge script syncs that data into task JSON files.

### Step 1: Set DATABASE_URL

```bash
# In .env.local
DATABASE_URL=postgresql://user:password@localhost:5432/your_database
```

### Step 2: Run database migrations

This creates the `token_usage` table and a `_migrations` tracking table:

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

Wraps any `claude` command and auto-captures token usage:

```bash
npm run dispatch -- my-task-id "Fix the authentication bug"
npm run dispatch -- refactor-api "Refactor the API layer" --model claude-sonnet-4-5
```

This:
1. Creates a task JSON file
2. Sets agent status to "working"
3. Runs `claude -p` with JSON output
4. Parses token counts from the response
5. Writes usage to both PostgreSQL and the task file
6. Sets agent back to "idle"

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

Works with or without PostgreSQL. Always writes to the task JSON file.

### Option C: sync-db bridge (from database)

If token usage is logged to the `token_usage` PostgreSQL table by other tools, `npm run sync` picks it up and attaches it to task JSON files automatically.

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

Checks in order: `claimed_by` → `assignee` → first `deliverables[].assignee`

### Usage array

Each entry in `usage` represents one API call. Fields:
- `inputTokens` (required) — also accepts `input_tokens`
- `outputTokens` (required) — also accepts `output_tokens`
- `cacheReadTokens` (optional) — also accepts `cache_read_tokens`
- `cacheWriteTokens` (optional) — also accepts `cache_write_tokens`
- `model` (optional) — e.g. "claude-opus-4", "claude-sonnet-4-5"
- `provider` (optional) — "anthropic", "openai", etc.
- `timestamp` (optional) — epoch milliseconds

---

## Dashboard Config File (Optional)

The bridge or your automation can write `tasks/dashboard-config.json` to customize the header dynamically:

```json
{
  "name": "My Swarm",
  "subtitle": "Mission Control",
  "repoUrl": "https://github.com/your/repo"
}
```

This overrides env vars and updates on the next API poll (30s).

---

## Agent Status File (Optional)

To show agents as "working" or "idle" on the agent strip, write `tasks/agents-status.json`:

```json
{
  "spark": "working",
  "scout": "idle",
  "critic": "working"
}
```

The `dispatch-task.sh` script manages this automatically. The sync-db bridge also writes it based on session heartbeats.

---

## Architecture

```
Data flow:

  [PostgreSQL] ──sync-db.ts──→ [tasks/*.json] ──data.ts──→ [/api/data] ──useSwarmData──→ [UI]
                                     ↑
  [dispatch-task.sh] ──log-usage.ts──┘
  [your scripts] ──log-usage.ts──────┘
```

Key files:
- `src/lib/data.ts` — Server-side: reads task JSON files, computes stats and token aggregates
- `src/app/api/data/route.ts` — API endpoint with auth + rate limiting
- `src/app/api/update/route.ts` — Dashboard self-update endpoint (git pull + restart)
- `src/lib/useSwarmData.ts` — Client hook, polls /api/data every 30s
- `src/components/TokenMetricsPanel.tsx` — Token usage charts (trend + model breakdown)
- `scripts/sync-db.ts` — PostgreSQL → task JSON bridge
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
| `npm run sync` | Sync PostgreSQL → task JSON files (one-time) |
| `npm run sync:watch` | Sync with file watch (re-runs on changes) |
| `npm run setup-db` | Run database migrations (creates token_usage table) |
| `npm run log-usage -- [args]` | Log token usage to DB + file |
| `npm run dispatch -- [task-id] [prompt]` | Run claude agent with auto-capture |
| `npm run lint` | Run ESLint |

---

## Verification Checklist

After setup, verify everything works:

1. `npx tsc --noEmit` — TypeScript compiles with zero errors
2. `npm run dev` — Dashboard loads at http://localhost:3000
3. Dashboard shows empty Kanban (normal if no task files yet)
4. Token Usage panel shows "No usage data yet" (normal)
5. Click a task card → modal appears centered, scrollable
6. Header shows your custom name (if configured)
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
| Modal appears at bottom of page | You're on an old version — `git pull` to get the flexbox modal fix |
| `next build` fails on `/_global-error` | Known Next.js 16 bug — does not affect functionality, ignore it |

---

## Security Notes

- API key auth protects `/api/data` and `/api/update` in production
- Rate limiting: 60 requests/min per IP on `/api/data`
- Path traversal protection on task file loading
- File size limit: 1MB per task JSON
- The `/api/update` endpoint runs `git pull --ff-only` (no force pulls, no rebase)
- Error messages are sanitized (no stack traces exposed)

---

## Do Not

- Do not edit `postcss.config.js` — it must stay as CommonJS (`.js`, not `.mjs`)
- Do not add `.env` files to git — they are gitignored (`.env.example` is the exception)
- Do not put non-JSON files in `OPENCLAW_TASKS_DIR` — only `*.json` is read
- Do not use `--force` with git operations from the update endpoint
- Do not commit `tasks/` contents — they are generated/ephemeral and gitignored
