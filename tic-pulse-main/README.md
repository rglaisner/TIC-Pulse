# TIC Pulse

Vite 6 + React 18 + TypeScript, Supabase, and Netlify Functions (scheduled fetchers).

## Prerequisites

- Node.js 20+
- Supabase project with schema applied (see `supabase/migrations/`)
- Netlify (or compatible) for deploy + scheduled functions

## Setup

1. Clone the repo and `cd` into this directory (the folder that contains `package.json`).
2. Copy `.env.example` to `.env` and fill in `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY`.
3. `npm ci`
4. `npm run dev` — app at http://localhost:5173

## Database migrations

Apply SQL in order:

1. `supabase/migrations/00000000000000_initial_schema.sql`

Use the Supabase SQL editor or CLI against a **fresh** project, then confirm RPCs and tables match production (see `docs/AGENT-TECHNICAL-AUDIT.md` and `schema/inventory.md`).

## Build & test

```bash
npm run typecheck
npm test
npm run build
npm run test:e2e
```

E2E starts the dev server with placeholder Supabase env unless you set `VITE_*` in the environment.

## Deploy (Netlify)

- **Build command:** `npm run build`
- **Publish directory:** `dist`
- **Functions:** `netlify/functions`
- Set function env vars in the Netlify UI: `SUPABASE_SERVICE_ROLE_KEY`, `VITE_SUPABASE_URL` (or `SUPABASE_URL`), `YOUTUBE_API_KEY`, `ANTHROPIC_API_KEY`, etc. (see `.env.example`).

Schedules are defined in `netlify.toml` (`fetch-gdelt`, `fetch-podcast`, `fetch-youtube`).

## Docs

- `docs/AGENT-TECHNICAL-AUDIT.md` — technical audit for agents
- `tic-pulse-refactor-plan.md` — historical refactor task graph
- `RELEASE-CHECKLIST.md` — pre-release verification
