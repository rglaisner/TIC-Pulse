---
name: TIC Pulse audit and refactor
overview: Produce a ruthless, agent-reusable technical audit document of the current TIC Pulse stack (Vite/React, Netlify Functions, Supabase), then execute a swarm-style, dependency-ordered refactor plan aligned with your Zod constitution and executing-plans workflow—culminating in a clean repo your friend can adopt and redeploy. Schema will be reverse-engineered from live Supabase (no dump available).
todos:
  - id: audit-doc
    content: Author docs/AGENT-TECHNICAL-AUDIT.md with all forensic sections + live Supabase facts after T0 inventory
    status: completed
  - id: swarm-plan-file
    content: Write tic-pulse-refactor-plan.md with full task IDs, depends_on, location, validation, parallel waves
    status: completed
  - id: t0-supabase-inventory
    content: "T0: Reverse-engineer production schema/RPCs/RLS via MCP/SQL; reconcile with all client+function table/RPC references"
    status: completed
  - id: t4-ts-zod
    content: "T4: Add TypeScript + Zod (env, API responses) per TYPESCRIPT-ZOD-RELIABILITY-CONSTITUTION"
    status: completed
  - id: t5-t6-functions
    content: "T5–T6: Unify Netlify handlers, fix fetch-podcast vs YouTube duplicate + schedule naming, dedupe cron config"
    status: completed
  - id: t7-t8-frontend-perf
    content: "T7–T8: Split App.jsx; pagination and query fixes; GDELT/YouTube cost caps"
    status: completed
  - id: t9-ci-tests
    content: "T9: Vitest + Playwright smoke + CI build"
    status: completed
  - id: t10-handoff
    content: "T10: README, .env.example, migration apply order, release checklist for friend"
    status: completed
isProject: false
---

# TIC Pulse: forensic audit + swarm refactor roadmap

## Context (what we already know from the repo)

- **Frontend:** Vite 6 + React 18, **JavaScript only** ([`tic-pulse-main/package.json`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\package.json)) — no TypeScript, no Zod, no test runner, no lint/format tooling.
- **Backend:** Netlify Functions under [`tic-pulse-main/netlify/functions/`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\netlify\functions) — mixed **`.js` / `.mjs`**, mixed **`export default` vs named `handler`**, duplicated scheduling config in places.
- **Data:** Supabase client in [`tic-pulse-main/src/supabase.js`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\src\supabase.js); SQL in repo is **incomplete** vs what the app actually uses.
- **Live app** ([ticpulse.netlify.app](https://ticpulse.netlify.app)): auth gate, dark “Pulse” / DM Sans + Playfair aesthetic — preserve this visual language during refactors.

---

## Part 1 — Deliverable: reusable audit document

**Create a single file** (suggested path: [`tic-pulse-main/docs/AGENT-TECHNICAL-AUDIT.md`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\docs\AGENT-TECHNICAL-AUDIT.md)) structured so any agent can pick it up without re-reading the whole tree. Recommended sections:

### 1. Executive verdict- One paragraph: fitness for production, biggest risks, cost drivers.

### 2. Repository and reproducibility

- **Schema gap:** Committed SQL only covers [`supabase-schema.sql`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\supabase-schema.sql) (articles + `article_engagement` + view + one RPC) and [`supabase-profiles.sql`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\supabase-profiles.sql) (profiles + trigger + engagement policy tightening). The app references **many other tables** with **no migrations in git:** `sources`, `videos`, `episodes`, `comments`, `user_engagement`, `user_streaks`, `user_profiles`, `brand_guidelines`, plus RPCs like `fetch_balanced_feed`, `increment_engagement`, `update_user_streak`, `increment_streak_counter`.
- **README drift:** Describes Phase 1 / 30-minute GDELT cadence; [`netlify.toml`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\netlify.toml) and [`fetch-gdelt.mjs`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\netlify\functions\fetch-gdelt.mjs) use **5-minute** schedule and a large multi-query/geo pipeline.
- **Nested folder:** Project lives under `tic-pulse-main/tic-pulse-main/` in the workspace — clarify root for CI and docs.

### 3. Netlify Functions — architecture and defects

| Finding | Evidence / impact |
|--------|-------------------|
| **Duplicate YouTube pipelines** | [`fetch-podcast.js`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\netlify\functions\fetch-podcast.js) and [`fetch-youtube.js`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\netlify\functions\fetch-youtube.js) are the same YouTube fetcher (headers and `[fetch-youtube]` logs inside `fetch-podcast.js`). **No podcast RSS ingestion in that file.** |
| **Schedule name mismatch** | `netlify.toml` schedules `[functions."fetch-podcasts"]` but the file is **`fetch-podcast.js`** → deployed name is almost certainly **`fetch-podcast`**, not `fetch-podcasts`. Likely **broken or no-op schedule**. |
| **Double scheduling** | `fetch-gdelt`: schedule in `netlify.toml` **and** `export const config = { schedule: ... }` in the function file — redundant; pick one source of truth to avoid confusion. |
| **Handler export inconsistency** | e.g. [`send-newsletter.js`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\netlify\functions\send-newsletter.js) uses **named** `handler`; others use **default** — hurts tooling and copy-paste safety. |
| **Env naming** | Functions use `VITE_SUPABASE_URL` server-side (works but violates separation of concerns; invites accidental client bundling mistakes in other stacks). |
| **Runtime cost** | `fetch-gdelt.mjs`: ~45 queries × 9 geos rotated, external RSS + **Anthropic** summarisation batch per run, every **5 minutes** — dominant **API bill and timeout risk**; needs explicit caps, idempotency, and observability. |
| **YouTube fetch** | [`getPlaylistItems`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\netlify\functions\fetch-youtube.js) paginates **entire playlist history** — **quota explosion** and long runtimes as channels grow; should bound by date or max pages. |

### 4. Frontend — structure and runtime

- **`App.jsx` is a monolith** (~1400+ lines per partial read): tabs (feed, discover, multimedia), newsletter, audio briefing, settings — **hard to test and reason about**.
- **Client loads up to 300 articles** then filters in memory ([`fetchArticles({ limit: 300 })`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\src\supabase.js), [`App.jsx`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\src\App.jsx)): **large payloads**, duplicate work vs DB-side filtering/pagination.
- **`fetchCategoryCounts`** ([`supabase.js`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\src\supabase.js)): `select("category")` on the feed **without limit** — **O(n) transfer** as DB grows.
- **Duplicate / stale comments** in code (e.g. “7-day” vs “14-day” freshness in [`App.jsx`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\src\App.jsx)) — signals lack of single source of truth.
- **No validation at boundaries:** all Supabase rows and `import.meta.env` are trusted — **direct conflict** with [`.cursor/rules/TYPESCRIPT-ZOD-RELIABILITY-CONSTITUTION.mdc`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\.cursor\rules\TYPESCRIPT-ZOD-RELIABILITY-CONSTITUTION.mdc).
- **Security / RLS:** [`supabase-profiles.sql`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\supabase-profiles.sql) tightens `article_engagement` to authenticated users, but **policies for `user_engagement`, `comments`, etc. are not in repo** — must be verified on live DB (risk of over-broad anon access or broken features).

### 5. Auth and data model alignment

- **Double profile creation:** [`AuthPage.jsx`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\src\AuthPage.jsx) upserts `profiles` after signup while SQL defines a **trigger** `handle_new_user` — redundant paths and possible race/conflict behavior.
- **Engagement model:** Per-article state in `user_engagement` + denormalised `article_engagement` — must document invariants (who may read/write liker lists, etc.).

### 6. Observability and operations

- **`console.log` / `console.error` only** in functions and client — no request IDs, no structured logs, no metrics dashboard story.
- **No automated tests** — regressions likely on any refactor.

### 7. Live app checklist (for humans/agents)

- Document: signup/login flow, main tabs, multimedia tabs, newsletter/audio flows — **note anything broken** (e.g. podcast schedule) after fixing function names.

### 8. “Fix later” backlog (prioritised)

- P0: schema-as-code, schedule/function naming, YouTube quota bounds, RLS audit.
- P1: TS + Zod, split `App.jsx`, server-driven pagination.
- P2: tests (Vitest + a few Playwright smoke tests per your rules), CI.

---

## Part 2 — Swarm-style refactor plan (per [`.cursor/rules/swarm-planner.mdc`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\.cursor\rules\swarm-planner.mdc))

**Output file:** `tic-pulse-refactor-plan.md` at the **project root you standardise** (recommend the inner [`tic-pulse-main/`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main) after removing double nesting or documenting it).

**Dependency graph (high level):**

```mermaid
flowchart TD
  T0[T0_InventoryLiveSupabase]
  T1[T1_AuditDoc]
  T2[T2_RepoHygiene]
  T3[T3_CanonicalMigrations]
  T4[T4_TS_Zod_Foundation]
  T5[T5_Functions_Unify]
  T6[T6_FixSchedules_Naming]
  T7[T7_Frontend_Split]
  T8[T8_Performance_API]
  T9[T9_Tests_CI]
  T10[T10_Handoff_Package]
  T0 --> T3
  T0 --> T1
  T1 --> T10
  T3 --> T5
  T3 --> T7
  T4 --> T7
  T4 --> T5
  T5 --> T6
  T6 --> T8
  T7 --> T8
  T8 --> T9
  T9 --> T10 T2 --> T4
```

### Task table (copy into `tic-pulse-refactor-plan.md` with full detail)

Each task must include: `id`, `depends_on`, `description`, `location`, `validation` (per swarm-planner). Summary:

| id | depends_on | What |
|----|------------|------|
| **T0** | [] | **Reverse-engineer live Supabase:** tables, views, functions, triggers, RLS policies, grants — via Supabase MCP / SQL introspection (`information_schema`, `pg_policies`, `pg_proc`). Produce `schema/inventory.json` or markdown snapshot + list of RPCs the app calls. |
| **T1** | [] | Write **AGENT-TECHNICAL-AUDIT.md** (Part 1 above, expanded with T0 facts). |
| **T2** | [] | **Repo hygiene:** single root folder story, `.env.example` aligned with all functions, Node20 pin, `engines` in package.json, optional `editorconfig`. |
| **T3** | [T0] | **Canonical migrations:** ordered SQL (or Supabase migration format) reproducing production; document how to apply to a fresh project. |
| **T4** | [T2] | **TypeScript + Zod:** `tsconfig` strict, Vite react-ts, Zod schemas for env (`client` vs `server`), `safeParse` at fetch boundaries, `z.infer` types — per constitution. |
| **T5** | [T3, T4] | **Functions package:** one folder, consistent `handler` export, shared `createSupabaseServiceClient()`, shared logger, Zod-validated event bodies where applicable; **delete or repurpose duplicate `fetch-podcast.js`**; implement **real podcast fetch** or remove dead schedule. |
| **T6** | [T5] | **Netlify.toml:** schedule keys match filenames; remove duplicate `config.schedule` in functions if toml is source of truth; document cron in README. |
| **T7** | [T3, T4] | **Split UI:** route-sized components/hooks (`useFeed`, `useEngagement`, multimedia hooks from [`useMultimedia.js`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\src\useMultimedia.js)); keep existing **inline aesthetic** (colors, fonts) unless a small a11y pass is agreed. |
| **T8** | [T6, T7] | **Cost/performance:** cap GDELT/Anthropic work per invocation; bound YouTube pagination; replace “load 300 + filter” with **paginated RPC or queries**; fix `fetchCategoryCounts` with SQL `GROUP BY` or materialized counts. |
| **T9** | [T8] | **Tests + CI:** Vitest for schemas/pure helpers; minimal Playwright smoke (login, feed visible); GitHub Actions `npm run build` + `npm test`. |
| **T10** | [T1, T9] | **Handoff package:** README “clone → env → migrate → deploy”, Netlify env matrix, **no secrets** in repo, tag release; optional `finishing-a-development-branch` checklist. |

**Parallel waves (for multi-agent runs):** Wave 1: T0, T1, T2 in parallel. Wave 2: T3 after T0; T4 after T2. Wave 3: T5, T7 after T3+T4. Then T6 → T8 → T9 → T10.

**External docs:** Before implementing T5/T6, refresh Netlify scheduled functions and Vite+TS docs (Context7 / web) for version-accurate config — per swarm-planner.

**Subagent review step:** After drafting `tic-pulse-refactor-plan.md`, run a review pass for missing dependencies, ordering errors, and edge cases (per swarm-planner §5); revise once.

---

## Part 3 — Alignment with [`.cursor/rules/executing-plans.mdc`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\.cursor\rules\executing-plans.mdc)

When execution starts (separate session):

1. **Announce** use of executing-plans; **load** `tic-pulse-refactor-plan.md`; **critically review** with the human; raise blockers before coding.
2. Use **isolated git worktree** (per that skill’s integration note) — **do not implement on `main` without explicit consent**.
3. For **each task:** `in_progress` → implement → run **validation** from the plan → `completed`; never skip verifications.
4. After all tasks: **finishing-a-development-branch** skill — tests green, merge/PR options.

---

## Part 4 — “New repo” packaging (friend handoff)

- **Clean tree:** `src/` (TSX), `netlify/functions/` (TS compiled or `.mts` with documented build), `supabase/migrations/`, `docs/AGENT-TECHNICAL-AUDIT.md`, `tic-pulse-refactor-plan.md` (historical).
- **Secrets:** document Netlify env vars (including `YOUTUBE_API_KEY`, `ANTHROPIC_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, anon URL/key for client).
- **Optional:** rename repo to `tic-pulse` and archive friend’s repo after first successful production cutover.

---

## Risks specific to “reverse-engineer only” schema

- **Drift:** Production may have hotfixes not reflected in introspection if permissions block full catalog reads.
- **Mitigation:** T0 includes explicit checklist of every `.from("…")` and `.rpc("…")` in [`src/`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\src) and [`netlify/functions/`](c:\Users\remyg\Projects\Toby News\tic-pulse-main\tic-pulse-main\netlify\functions); cross-check until100% covered.
