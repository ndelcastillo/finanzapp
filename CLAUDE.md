# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This repo has no build system, package manager, or test suite — it's two static files:

- `proyecto-villate.html` — a single-file web app ("Gastos en pareja" / "Shared couple expenses"), self-contained HTML/CSS/JS with no external JS dependencies (fonts are loaded from Google Fonts via `<link>`). This is the actual application.
- `DESIGN.md` — a standalone design-system reference document analyzing Apple's website (colors, typography, spacing, components). It documents a *different* aesthetic (photography-first, blue accent, SF Pro) than what `proyecto-villate.html` currently uses (warm editorial palette: Fraunces serif headings, Inter body, IBM Plex Mono for numbers, gold/clay/paper tones). Treat it as reference/inspiration material, not a spec already applied to the app — don't assume the two are in sync unless asked to reconcile them.

## Running / testing changes

There is no dev server, bundler, or test command. To verify changes, open `proyecto-villate.html` directly in a browser (or use the `run` skill to launch it) and exercise the UI manually — add an expense, toggle paid/unpaid, add income entries, change the split type, etc.

## Architecture of proyecto-villate.html

Everything lives in one file: `<style>` in the `<head>`, markup in `<body>`, and app logic in a single trailing `<script>`. There is no framework — DOM updates are done by re-rendering `innerHTML` strings from plain JS state.

**Persistence**: the app relies on a `window.storage.get/set(key, value, true)` API (Claude Artifacts' shared key-value storage runtime), not `localStorage`. State is shared with anyone who opens the artifact link — there's no per-user auth. Three storage keys:
- `expenses_v1` — array of expense objects
- `names_v1` — `{A, B}` display names for the two people
- `income_v2` — `{fx, entries: {A: [...], B: [...]}}`; a legacy scalar shape (`income_v1`) is migrated on load via `migrateIncome()`

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

- Keep everything in the one HTML file unless the user asks to split it up — that's a deliberate constraint of the Artifacts environment this was built for.
- Use `window.storage`, not `localStorage`, for anything that needs to persist.
- Follow the existing `A`/`B` person-keying convention rather than switching to names internally.
- `escapeHtml()` is used when interpolating user-entered text (`desc`, income `label`) into `innerHTML` — keep doing this for any new user-text interpolation to avoid XSS via the shared/multi-editor storage.
