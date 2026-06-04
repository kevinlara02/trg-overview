# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

**TRG Overview** is a private, single-page dashboard for a small restaurant group
(7 restaurants) that aggregates point-of-sale data from the **Clover POS REST API**.
It shows weekly sales, food-vs-bar breakdowns, hourly heatmaps, payment-method
analysis, expense tracking, and cross-restaurant comparisons.

The 7 restaurants: Toast Brea, Toast Downey, Little Toast, Story Whittier,
Story Anaheim, Benny and Mary's, The Benediction.

## Architecture — read this first

This is a **single self-contained `index.html` file** (~4000 lines). There is
**no build system, no framework, no package.json, no npm, no bundler, and no
backend server**. The entire app is:

- Vanilla JavaScript (no modules — everything hangs off `window.*`)
- Inline `<style>` block (minified CSS) in the `<head>`
- Chart.js 4.4 loaded from a CDN (`cdn.jsdelivr.net`)
- A large `EMBEDDED_DATA` object literal hardcoded inside a `<script>` tag

To run it: **just open `index.html` in a browser** (or serve the folder with any
static file server, e.g. `python3 -m http.server`). There is nothing to install,
compile, or transpile.

### Files

| File | Purpose |
|------|---------|
| `index.html` | The entire application — markup, styles, logic, and embedded data |
| `data.json` | Snapshot export of restaurant + sales + analytics data (matches the `EMBEDDED_DATA` shape) |
| `data (1).json` | A duplicate/older snapshot with the same key structure; not loaded by the app |
| `README.md` | Short human-facing summary |

`data.json` is **not fetched at runtime** — the app reads from the inline
`EMBEDDED_DATA` constant. The JSON files are reference snapshots/exports. If you
change the data model, keep `EMBEDDED_DATA`, `data.json`, and the README's
numbers consistent.

## Code layout inside index.html

Everything lives in one `<script>` block (roughly lines 40–4015). Key anchors:

- **`EMBEDDED_DATA`** (~line 42): the seed dataset. Top-level keys:
  `restaurants`, `colorMap`, `ingresosData`, `expenses`, `topItems`,
  `categoryData`, `detailData`, `orderStats`.
- **`AppState`** (~line 2561): the live, mutable state object —
  `{ restaurants, ingresosData, expenses, cloverData, currentTab }`.
- **Helpers** (~2710–2735): `window.fmt` (currency formatter), `window.getColor`,
  `window.getDates`, total/aggregation helpers, `window.syncAllData`,
  `window.refreshRestaurant`.
- **Render functions** (~2735–4010): one `window.render*` per tab, dispatched by
  `window.renderTab`. They build HTML strings and assign to `innerHTML`.
- **Clover fetch functions** (~3800+): `fetchPaymentsForRestaurant`,
  `fetchOrderSalesDataV2`, `fetchBasicStatsFromPayments`.
- **Bootstrap** (~line 4010): `DOMContentLoaded` → `renderApp()`.

### Rendering model

There is no virtual DOM and no reactive binding. The pattern is:

1. State lives in `AppState` and assorted `window.*` caches
   (`window.detailData`, `window.orderStats`, `window.categoryData`,
   `window.topItems`, `window.colorMap`).
2. `renderApp()` rebuilds the top-level shell (header + tab bar) into `#app`.
3. `switchTab(tab)` sets `window.activeTab` and calls `renderApp()`.
4. `renderTab()` dispatches to the active tab's `render*(el)` function, which
   writes an HTML string into `#main-content`.
5. Interactivity is wired through **inline `onclick`/`onchange` attributes** that
   mutate `window._*` filter flags or `AppState`, then call `renderTab()` /
   `renderApp()` to re-render.

The seven tabs: **Dashboard, Clover Income, Analytics, Expenses, Comparison,
Breakdown, Settings**.

### Data shapes (for reference)

- `ingresosData[]`: `{ date, restaurant, gross, discounts, tax, tips, net }` —
  one row per restaurant per day. (`ingresos` = Spanish for "income".)
- `expenses[]`: `{ date, vendor, restaurant, desc, amount, status, category }` —
  vendor invoices (US Foods, Webstaurant Store, Amazon, Other). Status values
  seen in code: `Paid` / `Pending`.
- `restaurants[]`: `{ name, merchantId, color, token }`.
- `detailData[name]`: `{ hourly, tenderTotals, orderTypeTotals, totalTips,
  totalTax, totalAmount, totalPayments }`.
- `orderStats[name]`: `{ totalOrders, avgTicket, tagSales[] }`.
- `categoryData[name][]`: `{ cat, total }`.
- `topItems[name][]`: `{ name, total, count }`.

Clover amounts come in **cents**; the fetch code divides by 100. `net` is
computed as `(totalAmount − totalTax) / 100`.

### Persistence

State is persisted to **`localStorage`**, not to disk/server. Keys:

- `trgExpenses` — expenses array
- `trgRestaurants` — restaurants array (so Settings shows tokens)
- `dashboardData` — holds `ingresosData` after a Clover sync

On load, expenses are read from `localStorage` falling back to `EMBEDDED_DATA`.

### Clover API integration

- `CLOVER_BASE = 'https://corsproxy.io/?url=https://api.clover.com/v3/merchants'`.
  Requests are routed through the **corsproxy.io** public CORS proxy because the
  app is a static page hitting Clover directly from the browser.
- Auth is `Authorization: Bearer <token>` per restaurant.
- `syncAllData()` loops every restaurant and calls
  `fetchPaymentsForRestaurant(name, token, merchantId)`, which pages through
  `/payments` (limit 200, offset paging) over a **rolling 7-day window**
  (`new Date()` minus 6 days → today), then re-renders.
- A sync is auto-triggered ~500ms after load
  (`setTimeout(() => window.syncAllData(), 500)`).

## Security note — API tokens

`EMBEDDED_DATA.restaurants` currently contains **live Clover API tokens and
merchant IDs hardcoded in `index.html`** (and in the `data*.json` snapshots).
The git history shows tokens being removed and then re-embedded for an
"automatic" public deployment.

Be deliberate here:

- **Do not** introduce *new* secrets or paste tokens into commits/PRs/comments.
- If asked to make the dashboard public/shareable, flag that the embedded tokens
  are exposed to anyone who can view the page, and prefer removing them or
  moving sync behind a server/proxy rather than embedding more credentials.
- Treat the existing tokens as already-compromised-by-design; don't echo them
  into chat, logs, or new files unnecessarily.

## Conventions

- **One file.** Keep changes inside `index.html` unless there's a strong reason
  to split. Match the existing style: `window.*` globals, inline event handlers,
  HTML built via template literals, minified inline CSS in `<head>`.
- **No dependencies/tooling.** Don't add npm, a bundler, a framework, or a build
  step unless explicitly requested — it would change the entire run model.
- **Currency:** always format money with `window.fmt(n)` (`$1,234.56`).
- **Colors:** per-restaurant colors come from `window.colorMap` / `getColor(name)`;
  reuse them for chart datasets and accents instead of hardcoding.
- **Dates:** use rolling/dynamic windows (`getDates()`, `new Date()` math). Avoid
  re-introducing hardcoded calendar dates — recent commits specifically replaced
  hardcoded dates with a dynamic rolling 7-day window.
- **Mixed language:** some identifiers/labels are Spanish (`ingresos` = income,
  `gastos` = expenses, `comparativo` = comparison). Function names like
  `renderIngresos`/`renderGastos` coexist with English equivalents.
- After editing, sanity-check by opening `index.html` in a browser — there are no
  automated tests, linters, or CI in this repo.

## Git / workflow

- Default branch: `main`.
- Commit messages in history follow a terse `Fix: <what>` / `<verb>: <what>`
  style. Keep them short and descriptive.
- Only open a pull request when the user explicitly asks for one.
- Do not commit or push new secrets.
