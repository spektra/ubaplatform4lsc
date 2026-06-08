# AGENTS.md — UBA Platform Redesign

**Status:** Pre-launch. No CI/CD. No tests. No linter/formatter. Manual verification via dev server only.

## Commands

| Command | Action |
|---|---|
| `npm run dev` | Vite dev server on `http://127.0.0.1:5173` |
| `npm run build` | Runs `tsc -b` then `vite build` |
| `npm run sheets:sync` | Dry-run Google Sheets importer (reads `.env.local`) |
| `npm run sheets:sync:write` | Write-mode sheets import (`--write`) |
| `npx supabase ...` | Supabase CLI v2.102.0 (pinned) |

No lint, test, format, typecheck-only, or codegen commands exist.

## OpenCode GUI MCP Config

This project has both a **project** `opencode.json` and a **global** config at `~/.config/opencode/opencode.jsonc`. The GUI merges both — a broken project config can break the entire GUI.

**Key rules** (see `docs/opencode-gui-mcp-config.md` for full guide):
- Local MCP `command` must be an **array**: `["/opt/homebrew/bin/npx", "-y", "package"]`
- Every MCP entry needs `"enabled": true`
- Use full path `/opt/homebrew/bin/npx` (Apple Silicon Homebrew)
- Check both project and global configs when troubleshooting
- Logs: `~/.local/share/opencode/log/`

## Stack

- React 19, TypeScript 6.x, Vite 8, Tailwind CSS v4
- **Tailwind v4:** uses `@tailwindcss/vite` plugin in `vite.config.ts`, NOT PostCSS. Custom theme in CSS `@theme` block in `src/styles.css`.
- React Router v7 (`react-router`, not `react-router-dom`)
- Supabase JS client v2 for auth + data (hosted: `ubaplatform-oceania`), Supabase CLI for local dev
- `ws` package is a runtime dep for Node-based Supabase client in scripts (browser uses native WebSocket)

## Env vars

- `VITE_*` vars (in `.env.example`) are public, shipped to browser via `import.meta.env`
- Non-`VITE_*` vars go in `.env.local` (gitignored) and are server-only — **never prefix secrets with `VITE_`**
- `src/lib/env.ts` reads `VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY`, exports `isSupabaseConfigured` and `ensureSupabaseEnv()`
- Google OAuth creds live outside repo at `~/.config/opencode/secrets/`

## Architecture

- **Entrypoint:** `src/main.tsx` → `<StrictMode>` → `<ErrorBoundary>` → `<NotificationProvider>` → `<BrowserRouter>` → lazy `<App>`
- **Data layer** (`src/lib/db.ts`): tri-state pattern — return module-scoped cache → static fallback if Supabase unconfigured → query Supabase → fallback on failure
- **Module-scoped caching** in `db.ts` — not React state. Call `clearTeamCache()` to invalidate
- **Static fallback data** lives in `src/data/` — all modules return typed arrays; player records return `[]` not fake data
- **Route pages:**
  - 21 lazy-loaded pages in `src/pages/` using **named exports** (`export function DashboardPage()`) imported as `lazy(() => import('./pages/...').then(m => ({ default: m.PageName })))`
  - Wrapped per-route in `<Suspense fallback={<PageFallback />}>`
- **Supabase types** are generated and committed at `src/lib/database.types.ts` (~1363 lines)
- **Migrations** live in `supabase/migrations/` numbered sequentially (0001-0018). New migrations follow the same pattern.
- **Bank→UC ledger pipeline:** migration 0018 adds `bank_promoted_at` / `bank_promoted_by` columns to `team_sheet_import_player_rows`, plus `promote_bank_to_uc_ledger` RPC that maps bank JSONB fields to `uc_ledger_events` rows. UI on ImportReviewPage.

## Conventions

- **ESM** (`"type": "module"` in package.json)
- **No path aliases** — all imports are relative
- **CSS:** `Bebas Neue` (display), `Inter` (body). Dark theme default, light via `html.theme-light` class. Design tokens use `--navy` through `--c-amber`, Shadcn‑style tokens (`--background`, `--foreground`, etc.), and `--app-*` aliases. Custom utility classes: `.premium-card`, `.premium-glass`, `.nav-glass`, `.page-title`, `.status-pill`, `.animate-soft-rise`, etc.
- **Typography:** Bebas Neue for display/headings (20px card titles, 36px hero, 22px logo), Inter for body. Letter-spacing 1.5px on card titles, 3px on hero.
- **Light mode:** Applied via `html.theme-light` class on `<html>`. Toggle persisted to `localStorage` key `uba-theme`. Old `--color-uba-*` Tailwind tokens are overridden in `html.theme-light` for backward compat. Hardcoded `text-white/*`, `bg-white/*`, `border-white/*` are replaced with CSS variable equivalents throughout all components and pages.
- **Inline CSS custom properties:** Use `cssProps()` from `src/lib/cssProps.ts` instead of `as React.CSSProperties` casts.
- **Icons:** Tabler Icons via CDN (`<i class="ti ti-*">`). No unicode/emoji in production code.
- **Shadow tokens:** Use `--shadow-menu`, `--shadow-nav`, `--shadow-dropdown`, `--shadow-jersey` CSS variables for theme-aware shadows, not hardcoded `shadow-black/*`.
- **Git:** conventional commits (`feat:`, `fix:`, `docs:`, optionally scoped)
- **OpenCode skill** for frontend design: load `design-taste-frontend` skill for UI work
- **Build verification:** CSS bundle ~71 kB. Build runs in ~150ms. No TypeScript errors.
- **Domain types:** `Team.id` = DB `slug` (set by `mapTeam`); `Player.id` = DB `slug ?? p.id` (set by `buildPlayer`). Use `.id` for all URL slugs, not a separate `.slug` field.

## Sheets importer

- `scripts/sync-team-sheets.mjs` — reads `docs/local/team_sheet_sources.json`, authenticates Google OAuth, fetches sheets, stages in Supabase
- Staging → promotion via `/import-review` page UI and `promote_sheet_import_player_row` RPC
- Tendencies set via `set_sheet_import_player_tendencies` RPC
- Bank→UC promotion via `promote_bank_to_uc_ledger` RPC on ImportReviewPage

## Gotchas

- **No tests exist.** Any code change is verified manually. Do not add test infrastructure unless asked.
- **TypeScript 6.x** is bleeding-edge; verify compatibility before adding any tooling
- **Supabase RLS mode:** `auto_expose_new_tables = false` in config (opt-in to new stricter mode)
- **Static data modules validate on import** (`src/data/league.ts` runs validation at module load)
- **`.env.local` is gitignored** — only `.env.example` is tracked
- **Site URL** in Supabase config: `http://127.0.0.1:5173`
