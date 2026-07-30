# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This repo has no build system, package manager, or test suite — it's two static files, deployed as-is via GitHub Pages:

- `index.html` — a single-file web app ("Gastos en pareja" / "Shared couple expenses"), self-contained HTML/CSS/JS with one external JS dependency (`@supabase/supabase-js`, loaded via CDN `<script>` — see Persistence below; fonts are loaded from Google Fonts via `<link>`). This is the actual application.
- `DESIGN.md` — a standalone design-system reference document analyzing Apple's website (colors, typography, spacing, components). It documents a *different* aesthetic (photography-first, blue accent, SF Pro) than what `index.html` currently uses (warm editorial palette: Fraunces serif headings, Inter body, IBM Plex Mono for numbers, gold/clay/paper tones). Treat it as reference/inspiration material, not a spec already applied to the app — don't assume the two are in sync unless asked to reconcile them.

## Running / testing changes

There is no dev server, bundler, or test command. To verify changes, open `index.html` directly in a browser (or use the `run` skill to launch it) and exercise the UI manually — add an expense, toggle paid/unpaid, add income entries, change the split type, etc. Since persistence now goes over the network to Supabase, testing multi-device sync means opening the file in two tabs/browsers and entering the same couple code in both.

## Architecture of index.html

Everything lives in one file: `<style>` in the `<head>`, markup in `<body>`, and app logic in a single trailing `<script>`. There is no framework — DOM updates are done by re-rendering `innerHTML` strings from plain JS state.

**Persistence**: backed by Supabase (Postgres + REST + Realtime), not `localStorage` or `window.storage` (this app previously ran inside Claude Artifacts, where `window.storage` was Artifacts' own shared KV runtime — that environment no longer applies here). Every couple is one row in a `couples` table, keyed by a shared "código de pareja" the two people agree on (no email/password — see `initCouple()`/`enterCouple()`). The `store.get(key)`/`store.set(key, value)` object (search `const store = {` in the script) still exposes the original key-based interface so the ~15 call sites elsewhere didn't need to change; internally it now maps logical keys to `couples` columns and does Supabase `select`/`update` calls:
- `expenses_v1` → `expenses` column — array of expense objects
- `names_v1` → `names` column — `{A, B}` display names for the two people
- `income_v2` → `income` column — `{fx, entries: {A: [...], B: [...]}}`; a legacy scalar shape is migrated on load via `migrateIncome()`
- `fixed_v1` → `fixed` column, `fixedincome_v1` → `fixed_income` column
- the one-time migration flags (`fixed_m1`, `fixed_m2`, `fixed_m3`, `fixedincome_m1`) collapse into a single `migration_flags` jsonb column instead of separate rows

A Supabase Realtime subscription (`subscribeCoupleRealtime()`) pushes every `UPDATE` on the couple's row to all other open tabs/devices with the same code, which is what makes cross-device sync work without polling. Row Level Security on `couples` allows any `anon` client to read/write — the couple code itself is the only access control, mirroring the no-auth model the app already had. The Supabase project URL and anon key are hardcoded near the top of the script (`SUPABASE_URL`, `SUPABASE_ANON_KEY`) — this is intentional and safe: Supabase anon keys are meant to be public, security comes from RLS policies, not from hiding the key.

**People model**: the two participants are always referred to internally as `'A'` and `'B'` (not by name) — expense `paidBy`, split percentages (`pctA`, with B implicitly `100 - pctA`), and income `entries` are all keyed this way. Display names (`names.A`/`names.B`) are cosmetic only and looked up at render time.

**State → render flow**: every mutation (add/delete/toggle expense, add/remove income, change names or FX rate) follows the same pattern: mutate the in-memory object → persist via the relevant `save*()` function (fire-and-forget async, wrapped in try/catch) → call `render()` (or `renderIncomeLists()` for income-specific UI) to redraw affected DOM sections from scratch. There is no diffing — sections are fully re-stringified and reassigned to `innerHTML`.

**Key derived calculations** (in `render()`'s call graph):
- `renderBudget()` — "disponible para gastar": each person's income minus their split share of expenses flagged `essential` and dated in the current month.
- `renderCharts()` → `renderBalance()` — computes net debt between A and B across *all* expenses (not just current month) by summing each person's owed share vs. what they actually paid.
- `incomePctA()` — proportional income split (used when an expense's `splitType === 'income'`), converts any USD income entries to ARS using the manually-entered `fx` rate before comparing.
- Recurring expenses: marking a recurring expense paid (`togglePaid`) auto-generates the next month's copy via `addMonths()`, and bumps `cycleCount`; if `adjustPct > 0` and the new cycle is a multiple of 3, the amount is escalated by that percentage.

**Charts**: category and payer breakdowns are rendered as hand-rolled SVG pie charts (`pieSvgHtml`/`arcPath`/`polarToCartesian`) — no charting library.

**i18n**: all user-facing strings are hardcoded Spanish (Argentine locale — `es-AR` for `Intl.NumberFormat`/date formatting, ARS as the default currency). There's no translation layer; don't add one unless asked.

## Conventions to preserve when editing

- Keep everything in the one HTML file unless the user asks to split it up — GitHub Pages serves it as a static file with no build step.
- Use the `store.get`/`store.set` abstraction (backed by Supabase) for anything that needs to persist — don't reach for `localStorage` as a source of truth; it's only used to remember which couple code this device last used (`couple_code` key), not for actual data.
- Follow the existing `A`/`B` person-keying convention rather than switching to names internally.
- `escapeHtml()` is used when interpolating user-entered text (`desc`, income `label`) into `innerHTML` — keep doing this for any new user-text interpolation to avoid XSS via the shared/multi-editor storage.
- Every `couples` row is shared by everyone who knows that couple's code — there is still no per-person auth within a couple, only isolation *between* couples.
