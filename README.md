# TS Raceroom

Desktop prep tool for [Touge Shakai](https://touge-shakai.com/) team battles — pull public leaderboard data, set up your crew, scout rivals, plan matchups, and run battles round by round without digging through the website every time.

Early days, soft launch. Feedback welcome, especially if something looks off on your track or car combo.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/nephilimzero)

## Download (Windows)

The easiest way to get started is the installer from **[GitHub Releases](https://github.com/GlitchBase/TSRaceroom/releases)**:

1. Open the latest release (e.g. `v0.7.0`)
2. Download **`TS-Raceroom-Setup-x.x.x.exe`**
3. Run the installer

The desktop app checks for updates from **Settings → App updates** (packaged builds only).

## What's in the app today

- **My Team** — roster with drivers, preferred cars/direction, home & backup maps, team logo
- **Opponents** — rival teams; pin one as your active **Challenging** opponent
- **Matchup planner** — pairings, gap callouts, battle brief, saved matchups, printable/copyable battle sheet
- **Rounds** — live score tracking, home/away venues, map bans, tiebreaker flow, coin toss
- **Race Prep** — battle track, weather, and quick links into matchup + rounds
- **Leaderboard** — browse times with your team and pinned drivers highlighted
- **Dashboard** — recent improvements and cache status for your roster
- **Steam sign-in** — auto-add yourself to the roster and highlight your times (desktop app)
- **Team import/export** — share rosters with teammates via file or clipboard
- **NIGHTSHIFT** — link to the Touge Shakai community Discord from the sidebar

## Run from source

For development or if you prefer to run the web UI locally:

```bash
npm install
npm run dev
```

Open http://localhost:5173

To test on another device on your network:

```bash
npm run dev:lan
```

Use the Network URL from the terminal. Steam sign-in and file storage only work in the desktop app.

### Desktop app (Electron)

```bash
npm run electron:dev
```

Build a Windows installer locally:

```bash
npm run electron:build
```

Output goes to `release/`.

## Data

Leaderboard data comes from the game's public API (same as the website):

- `/getCarTrackList/` — tracks and cars
- `/getRecords/` — leaderboard entries

Your team, opponents, matchups, and cache are stored locally on your device. A full leaderboard cache refresh is manual from Settings; your team data refreshes daily.

## Stack

React, Vite, React Router, Electron

## Cloud sync (Vercel + Supabase) [optional]

This repo can be deployed to Vercel and extended with a Supabase Postgres database to enable
Steam-only sign-in and cross-device sync (personal workspace first; team workspaces later).

**Live web app:** https://ts-raceroom.vercel.app

### Environment variables (Vercel)

- `SUPABASE_URL` — e.g. `https://opgznlgksjwdhdsbmgqk.supabase.co`
- `SUPABASE_SERVICE_ROLE_KEY` — server-only (Supabase → Project Settings → API → `service_role`)
- `SUPABASE_ANON_KEY` — publishable anon key (Realtime client + token endpoint)
- `SUPABASE_JWT_SECRET` — JWT secret (Settings → API) for Realtime auth tokens
- `SESSION_SECRET` — long random string
- `CRON_SECRET` — long random string (Vercel Cron auth for leaderboard poller)
- `VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY` — same URL/anon key for web build (Realtime)

See `.env.example` for placeholders.

### Setup steps

1. Create a Supabase project and run the SQL in `supabase/schema.sql`
2. Apply `supabase/migrations/20260710_leaderboard_watch.sql` for server-side PB watch tables
3. Add the environment variables above to Vercel (Production)
4. Deploy — `npx vercel deploy --prod` or push to the linked GitHub repo

### Server PB watch (Phase 1–2)

Vercel Cron calls `/api/cron/poll-leaderboard` daily at **12:00 UTC** on Hobby plans (manual curl any time). Pro can use `*/15 * * * *` for 15-minute polling.

**Phase 1** — cron poller writes `board_snapshots` and `time_events`.

**Phase 2** — authenticated read APIs for signed-in users:

- `GET /api/watch/events?since=&limit=&side=&types=`
- `GET /api/watch/snapshots?side=`
- `GET /api/watch/status`

Client helpers: `src/services/serverWatch.js` (requires Steam sign-in / cloud session).

**Phase 4** — Supabase Realtime live PB toasts (apply `supabase/migrations/20260710_watch_realtime.sql`, set anon + JWT secret env vars).

Manual cron test after deploy:

```bash
curl -H "Authorization: Bearer YOUR_CRON_SECRET" https://ts-raceroom.vercel.app/api/cron/poll-leaderboard
```

Rollout plan: `docs/server-watch-phases.md`

### Test cloud sync

1. Open https://ts-raceroom.vercel.app/settings/general → **Sign in with Steam**
2. Go to **Settings → Manage Data → Cloud sync** → Push / Pull
3. Or edit team/opponents — web auto-syncs after sign-in

**Local full-stack dev:** run `npm run dev:web` (Vercel dev + API routes). Plain `npm run dev` still proxies leaderboard `/api` to touge-shakai; auth/sync routes are excluded and require `dev:web`.

Notes:
- Do **not** commit DB passwords/keys.
- The leaderboard cache is rebuildable and stays local-first; sync focuses on team/profile docs.
- RLS is disabled on sync tables because only the Vercel API uses the service role (no anon key in the browser).
