# GEAP Brokerage Workspace POC — UI Implementation Plan

> **Purpose of this document.** This is the only artifact available to rebuild the **front-end (UI)** of the GEAP E\*TRADE-style proof-of-concept. It describes what the app looks like, how it behaves and how it is put together, in enough detail to rebuild it from nothing. Verbatim data, prompts, demo copy and stylesheets are in the **Appendices** (§A–§H). Treat the appendix content as the source of truth wherever prose and appendix disagree.
>
> **Reference experience:** `http://localhost:3000/?demo=true`. Demo mode renders the whole workspace with synthetic data and canned "AI" answers, with no brokerage login and no Gemini API key.
>
> **Out of scope:** the Python ADK agent backend (`geap_agent/`), the deployment tooling (Agent Engine, Cloud Build, GitHub Actions) and the presentation scripts. The Node server appears only as far as the UI needs it (§3).

---

## Table of contents

1. [Product summary](#1-product-summary)
2. [Tech stack & hard constraints](#2-tech-stack--hard-constraints)
3. [Minimal server contract (what the UI needs)](#3-minimal-server-contract-what-the-ui-needs)
4. [Project layout](#4-project-layout)
5. [Boot sequence, script loading & lazy modules](#5-boot-sequence-script-loading--lazy-modules)
6. [Data visibility modes: demo / live / disconnected](#6-data-visibility-modes-demo--live--disconnected)
7. [Global state & persistence](#7-global-state--persistence)
8. [Data layer](#8-data-layer)
9. [Router & navigation](#9-router--navigation)
10. [App shell (index.html)](#10-app-shell-indexhtml)
11. [Views](#11-views)
12. [Chat assistant (the core AI experience)](#12-chat-assistant-the-core-ai-experience)
13. [Agent registry, tools & auto-routing](#13-agent-registry-tools--auto-routing)
14. [Gemini client](#14-gemini-client)
15. [Context engine](#15-context-engine)
16. [Governance & guardrails matrix](#16-governance--guardrails-matrix)
17. [Design system & styling](#17-design-system--styling)
18. [Accessibility](#18-accessibility)
19. [Testing strategy](#19-testing-strategy)
20. [Build order / milestones](#20-build-order--milestones)
21. [Demo walkthrough acceptance script](#21-demo-walkthrough-acceptance-script)
22. [Known quirks & recommended fixes](#22-known-quirks--recommended-fixes)
- Appendices A–H (verbatim data, prompts, copy, CSS)

---

## 1. Product summary

A framework-free single-page app (SPA) that mocks a retail brokerage customer portal (purple E\*TRADE-like look) and layers **GEAP (Gemini Enterprise Agent Platform) AI features** on top:

| Area | Route | What it shows |
|---|---|---|
| Complete View dashboard | `/`, `/accounts` | Welcome + totals, a "GEAP Agent Briefing" card, an expandable AI Insights strip, account cards, watch list, side-rail summary cards, disclosures |
| Portfolios | `/accounts/portfolios` | 7 tabs (Positions, Allocation, Performance, Historical Value, Gains & Losses, Risk Assessment, Estimated Income), an account selector, donut charts, an AI risk diagnostic |
| Planning & Retirement | `/planning` | AI Retirement Advisor hero, projection stats, tool tiles, safeguards |
| AI Insights Hub | `/ai-insights` | AI advisory fleet cards, proactive recommendations, a What-If trade sandbox with an animated multi-agent stepper |
| AI Settings | `/ai-settings` | Customer guardrails: access mode, trade execution mode, PII scrubbing, info-source whitelist (persisted to localStorage) |
| Agent Studio (admin) | `/agent-studio` | Agent list, flow diagram, test console, config panel, "Create New Agent" modal |
| Agent Fabric (admin) | `/agent-fabric` | Fleet metrics, agent cards, live activity log, routing map, governance toggles |
| Chat assistant | floating, on every page | Multi-agent chat with auto-routing, PII shield, safety filter, disclaimers, escalation, provenance cards, interactive widgets (rebalance ticket, retirement charts, portfolio analyzer, orchestration graph), voice in/out, animated "neural" avatar |
| Placeholders | ~15 other routes | A generic "wired into the mock navigation" page |

Every value is synthetic. The footer states "Internal GEAP mock POC. All data is synthetic."

---

## 2. Tech stack & hard constraints

- **No front-end framework.** Plain ES2020+ JavaScript, one global module per file built with the IIFE/revealing-module pattern (`const X = (() => { …; return {…}; })();`), with HTML produced by **template literals** and assigned via `innerHTML`.
- **Globals are part of the contract.** Modules talk to each other through globals: `state`, `mockData`, `render`, `routeTo`, `selectedAccount`, `renderDisclosures`, `moneyToneClass`, `escapeHtml`, `isShowDataEnabled` (also on `window`), `triggerChatQuery` (on `window`), `GeapApp`/`App`, `BrokerageData`, `Charts`, `ContextEngine`, `AgentManager`, `GeminiAPI`, `Chat`, `InsightsEngine`, `AIInsightsPage`, `AgentStudio`, `AgentFabric`, `renderPortfolios`, `renderPlanning`, `renderAISettings`, `bindAISettingsEvents`. Inline `onclick="Chat.xxx()"` handlers depend on these names.
- **Classic `<script defer>` tags, not ES modules.** The one exception is the Gemini SDK, pulled in through dynamic `import('https://esm.run/@google/genai')`.
- **Fonts:** the system stack `Arial, Helvetica, sans-serif`, 14px base. No web fonts are loaded; the canvas chart code mentions "Inter" and "JetBrains Mono" but falls back to the system stack.
- **Server:** Node ≥ 18 with Express 4 for static files and SPA fallback, `compression` (gzip) and `express-session`. Optional: `dotenv`, `oauth-1.0a` (only for the live-brokerage feature).
- **Testing:** `node --test` for unit and integration tests, plus `puppeteer-core` for E2E.
- **Browser APIs used:** `history.pushState`/`popstate`, `localStorage`, `sessionStorage`, `fetch`, Web Speech API (`SpeechRecognition`/`webkitSpeechRecognition`, `speechSynthesis`), Canvas 2D, and CSS `conic-gradient`.

---

## 3. Minimal server contract (what the UI needs)

The SPA has to be served so that deep links work. Implement `server.js` (Express) with:

1. **Gzip compression** on every response.
2. **Startup CSS minification (optional but done in the original).** For every `public/css/*.css` that does not already end in `.min.css`, write a sibling `*.min.css` using these regex passes: strip `/* … */`, collapse whitespace, remove spaces around `{}:;,`, replace `;}` with `}`. **The HTML and JS reference only the `.min.css` files.** If you skip this step, check in the minified files or reference the plain `.css` files instead.
3. **Static files** from `public/` with `extensions: ["html"]`, `etag: true`, and `maxAge` of `1h` in production and `0` otherwise. Call `app.disable("x-powered-by")`.
4. **No-cache headers** (`Cache-Control: no-store…`, `Pragma: no-cache`, `Expires: 0`) on `/api/*` and `/auth/*`.
5. **SPA routes.** `GET` on each of the following returns `public/index.html` with status 200:
   `/`, `/accounts`, `/accounts/portfolios`, `/accounts/watch-lists`, `/accounts/orders`, `/accounts/balances`, `/accounts/activity`, `/accounts/banking`, `/accounts/tax-center`, `/accounts/documents`, `/accounts/dividend-reinvestment`, `/accounts/open-account`, `/pay-transfer`, `/trading`, `/markets-ideas`, `/at-work`, `/planning`, `/what-we-offer`, `/support`, `/alerts`, `/documents`, `/profile`, `/search`, `/agent-studio`, `/agent-fabric`, `/ai-insights`.
   (`/ai-settings` is not in the list, so it is served by the 404 fallback, which still loads the SPA. Add it to the list in the rebuild.)
6. **Fallback.** Any other path returns `index.html` with status **404**.
7. **APIs the UI calls:**

| Endpoint | Used by | Demo-mode requirement |
|---|---|---|
| `GET /api/etrade/status` → `{ connected: boolean, env: "sandbox"\|"production" }` | `BrokerageData.checkEtradeStatus()` on every page load | **Required** (return `{connected:false, env:"sandbox"}` when there are no credentials). If it fails the UI logs the error and carries on. |
| `GET /api/etrade/disconnect` → `{success:true}` | "Log Off" link | Optional |
| `GET /api/etrade/accounts`, `/api/etrade/portfolio/:key`, `/api/etrade/balances/:key`, `/api/etrade/transactions/:key` | Live mode only (§8.3) | Optional. Return 500 with `{error}` when not connected. |
| `GET /auth/etrade` (opens the OAuth 1.0a request-token flow in a popup) and `GET /auth/etrade/callback?pin=…` | "Log On" link | Optional |
| `GET /api/search?q=…&domains=a.com,b.com` → `[{title, content, source}]` (top 5) | `searchFinancialInfo` tool when real Gemini is used | Optional. The original proxies Google News RSS (`https://news.google.com/rss/search?q=<q> (site:a OR site:b)`), parses `<item>` title/link/pubDate/source with regex, decodes entities, normalizes the source name to Yahoo Finance, Bloomberg, Reuters or Morningstar, and returns `content: "Link: …\nPublished: …\n[Source: …]"`. |

Port: `process.env.PORT || 3000`. `npm start` runs `node server.js`; `npm run dev` runs `node --watch server.js`.

---

## 4. Project layout

```
public/
  index.html                 # SPA shell, critical inline CSS, header/nav, chat launcher, placeholder agent panel
  css/
    styles.css               # base E*TRADE look + shared GEAP components (eager, via preload)
    chat.css                 # chat panel, neural avatar, widgets, GEAP design tokens (lazy)
    insights.css             # AI Insights Hub page (lazy)
    settings.css             # AI Settings page (lazy)
    agent-studio.css         # Agent Studio (lazy)
    agent-fabric.css         # Agent Fabric (lazy)
    *.min.css                # generated or checked in (what HTML/JS actually reference)
  js/
    data.js                  # BrokerageData: synthetic holdings, quotes, logs, formatters, live-mode loader
    charts.js                # Charts: canvas line/donut/sparkline/bar (available, lightly used)
    context.js               # ContextEngine: structured context payload for prompts
    agents.js                # AgentManager: agent registry, tool implementations, auto-router
    gemini.js                # GeminiAPI: @google/genai streaming + function calling + fallbacks
    app.js                   # state, mockData, router, dashboard, shell events, GeapApp helpers
    portfolios.js            # renderPortfolios + tabs (lazy)
    planning.js              # renderPlanning (lazy)
    insights.js              # InsightsEngine (dashboard strip) + AIInsightsPage (lazy)
    settings.js              # renderAISettings + bindAISettingsEvents (lazy)
    agent-studio.js          # AgentStudio (lazy)
    agent-fabric.js          # AgentFabric (lazy)
    chat.js                  # Chat (lazy, loaded on first open)
server.js
package.json
tests/{unit,integration,e2e}/
```

---

## 5. Boot sequence, script loading & lazy modules

### 5.1 `index.html` head
1. An inline `<script>` defines two idempotent loaders:
   - `window.ensureScript(src, cb)` strips the query string to get `cleanSrc`. If `window.loadedScripts` (a Set) already contains it, or a `script[src^="cleanSrc"]` exists, it calls `cb()` immediately. Otherwise it appends a `<script>` to `body`. On `onload` it adds the path to the set and calls `cb`; `onerror` also calls `cb`.
   - `window.ensureStyle(href, cb)` does the same with `<link rel="stylesheet">` appended to `head`, tracked in `window.loadedStyles`.
2. An inline `<style>` with **critical CSS**: the `:root` tokens and every rule up to the secondary nav (a copy of the top of `styles.css`), plus `@keyframes geapPulse` (opacity 0.6 → 0.3 → 0.6) for the skeleton placeholder.
3. The non-blocking main stylesheet: `<link rel="preload" href="/css/styles.min.css?v=0.1.1" as="style" onload="this.onload=null;this.rel='stylesheet'">`, plus a `<noscript>` fallback.

### 5.2 Eager scripts (end of body, all `defer`, in this order)
`/js/data.js`, `/js/charts.js`, `/js/context.js`, `/js/agents.js`, `/js/gemini.js`, `/js/app.js` (each with `?v=0.1.1`).

### 5.3 Lazy modules (loaded by `render()` on first visit)
| View / trigger | Style | Script | Global checked |
|---|---|---|---|
| portfolios | — | `/js/portfolios.js?v=0.1.0` | `renderPortfolios` |
| planning | — | `/js/planning.js?v=0.1.0` | `renderPlanning` |
| ai-insights | `/css/insights.min.css?v=0.1.0` | `/js/insights.js?v=0.1.0` | `AIInsightsPage` |
| agent-studio | `/css/agent-studio.min.css?v=0.1.0` | `/js/agent-studio.js?v=0.1.0` | `AgentStudio` |
| agent-fabric | `/css/agent-fabric.min.css?v=0.1.0` | `/js/agent-fabric.js?v=0.1.0` | `AgentFabric` |
| ai-settings | `/css/settings.min.css?v=0.1.0` | `/js/settings.js?v=0.1.0` | `renderAISettings` |
| Dashboard AI strip expanded | `/css/insights.min.css?v=0.1.0` | `/js/insights.js?v=0.1.0` | `InsightsEngine` |
| Chat launcher click, or `triggerChatQuery` | `/css/chat.min.css?v=0.1.0` | `/js/chat.js?v=0.1.0` | `Chat` |

Lazy-load pattern inside `render()`: if the global is undefined, set `app.innerHTML` to a centered `Loading <Name>...` message (`padding:2rem; text-align:center; color:var(--muted); font-size:1.1rem; font-weight:500`). Then call `ensureStyle` and `ensureScript`, and in the callback call `render()` again **only if `getMeta().view` still matches**, which guards against the user navigating away mid-load.

### 5.4 Initial render (bottom of app.js)
```text
isEtradeConnectedSession = sessionStorage.etrade_connected === "true"
if (!isEtradeConnectedSession) render()          // instant paint (demo or skeleton)
if (in browser) BrokerageData.checkEtradeStatus()
   .then(() => { updateEtradeUI(); if (liveMode || isEtradeConnectedSession) render(); })
   .catch(() => { if (isEtradeConnectedSession) render(); })
```
Before JS runs, `<main id="app">` contains a **static skeleton**: a gray section (`padding:24px 0; min-height:80vh`) with a 280×32 bar, then a 2fr/1fr grid. The left column holds 180px and 340px white cards; the right holds 120px and 240px cards. All are `#e2e8f0` bordered, 8px radius, and animated with `geapPulse 1.5s infinite`.

---

## 6. Data visibility modes: demo / live / disconnected

`isShowDataEnabled()` (in app.js, exported on `window`) returns **true** when any of these holds:
- **Live:** `BrokerageData.getLiveMode()` is true (real brokerage connected).
- **Test:** `window.__e2e_test_active__`, or Node with `NODE_ENV==="test"`, `PORT==="3099"` or a `test` argv.
- **Demo:** `window.location.search.includes("demo=true")`.

Behavior:
- **false (disconnected, no demo):** every data card renders **skeleton placeholders** using the `.geap-skeleton-text`, `.geap-skeleton-title`, `.geap-skeleton-circle`, `.geap-skeleton-line` and `.geap-skeleton-card` classes (gray `#e2e8f0` blocks, pulsing). `syncEtradeToMockData()` empties `mockData.accounts`, sets the totals to `$0.00` and sets `morganStanleyAccount` to null. `BrokerageData.checkEtradeStatus()` also clears holdings, transactions, sectors and account numbers when neither test nor demo is active. Once data is "shown" with zero brokerage accounts, the dashboard and portfolios display an **"E\*TRADE Connection Required"** empty state: a 🔌 icon, a purple heading, a muted paragraph, and a `🔌 Connect E*TRADE Account` button (class `button etrade-connect-prompt-btn`).
- **true + demo/test:** on every `render()`, `syncEtradeToMockData()` deep-copies `originalMockData` back into `mockData` (user, totals, accounts, morganStanleyAccount). If "Include external accounts" is on, it adds the Morgan Stanley account's assets and day gain to the totals. A `selectedAccountId` starting with `etrade_` is reset to `core`.
- **true + live:** see §8.3.

> **Important (see §22):** in the original, `routeTo()` pushes clean paths, so `?demo=true` is lost after the first SPA navigation and later views fall back to skeletons. In the rebuild, make demo mode **sticky**: on boot, if `?demo=true` is present, set `sessionStorage.geap_demo = "1"` and have `isShowDataEnabled()` also check that flag. Optionally keep `?demo=true` on pushed URLs.

---

## 7. Global state & persistence

`app.js` defines the global `state`:

```js
const state = {
  route: location.pathname,
  selectedAccountId: "core",
  allocationMode: "asset",               // "asset" | "sector"
  portfolioTab: new URL(location.href).searchParams.get("tab") || "allocation",
  alertTab: "all",                       // "all" | "account" | "symbol"
  expandedAccounts: new Set(["elcie"]),  // account cards showing the movers table
  visibleNumbers: new Set(),             // accounts with the full account number revealed
  aiDetailsExpanded: false,              // dashboard AI strip open/closed
  includeExternalAccounts: false,
  governance: { safety:true, disclaimers:true, pii:true, rateLimiting:true, escalation:true, audit:true },
  aiSettings: loadFromLocalStorage("geap_ai_settings") ?? {
    allowedSources: { yahoo:true, sec:true, bloomberg:true, reuters:true, morningstar:true,
                      private_holdings:true, private_transactions:true, private_tax:true },
    accessMode: "read-write",            // "read-only" | "read-write"
    tradeExecutionMode: "execute",       // "execute" | "draft" | "never"
    piiScrubbingEnabled: true
  }
};
```
Wrap the localStorage read in try/catch and guard `typeof localStorage`.

| Storage | Key | Value / purpose |
|---|---|---|
| localStorage | `geap_ai_settings` | JSON of `state.aiSettings` (written by the AI Settings page) |
| localStorage | `gemini_api_key` | Gemini API key (written by the API key modal) |
| sessionStorage | `etrade_connected` | `"true"` while a live session is believed active (skips the first paint to avoid a flash) |
| sessionStorage | `geap_demo` | **New in the rebuild**: the sticky demo flag |

`state.governance` lives in memory only and is toggled from Agent Fabric.

---

## 8. Data layer

### 8.1 `mockData` (app.js): dashboard presentation data
Strings are pre-formatted (`"$18,427.55"`). The structure:
- `user` `{firstName:"Ken", lastLogin, refreshedAt}`
- `totals` `{totalAssets:"$48,912.74", dayGain:"+$318.22"}`
- `accounts[]`: six accounts with ids `main`, `core`, `elcie`, `checking`, `individual` and `stock`. Each has `id, label, type, masked, full, netValue, dayGain, dayGainTone, available, cash, beneficiary, expanded`, plus `subtitle`, `allocation` (core), `movers[]`, `bank:true` with `totalBalance` (checking) and `benefit` (stock).
- `sidePanels.launchPad[]` (5) and `sidePanels.alerts[]` (5, with `category` set to `symbol` or `account`)
- `watchList[]` (RIVN, MBLY)
- `morganStanleyAccount` `{name:"Premier CashPlus - 7306", assets:"$8,104.31", dayGain:"$0.00", dayGainPct:"0.00%"}`

`originalMockData = JSON.parse(JSON.stringify(mockData))` is taken right after the definition. **The verbatim object is in Appendix A.**

### 8.2 `BrokerageData` (data.js): the numeric portfolio engine
An IIFE that exposes:

| Member | Description |
|---|---|
| `account` | `{name:'Jonathan Doe', accountNumber:'****4821', accountType, cashBalance:14869.92, buyingPower:29739.84, totalValue:247832.15, dayChange:1243.67, dayChangePercent:0.5043, totalReturn:47832.15, totalReturnPercent:23.92, totalCost:200000, openDate}` |
| `holdings[]` | 12 positions (AAPL, GOOGL, MSFT, AMZN, NVDA, TSLA, VTI, QQQ, BND, GLD, JPM, JNJ). Each has `symbol,name,shares,avgCost,currentPrice,sector,dayChange,dayChangePct`, plus derived `marketValue=shares*price`, `costBasis=shares*avgCost`, `totalReturn`, `totalReturnPct=(price-avg)/avg*100`, `dayChangeTotal=shares*dayChange`, and `weight` (% of total holdings market value). All derived values use `+x.toFixed(2)`. |
| `sectorAllocation` | `{ [sector]: {value, weight, color} }`, aggregated from holdings, with HSL colors per sector (Appendix B) |
| `watchlist[]` | 8 symbols |
| `marketIndices[]` | S&P 500, NASDAQ, DOW, Russell 2K, BTC/USD, 10Y Yield, VIX, Gold |
| `transactions[]` | 12 rows `{date,type(BUY/SELL/DIVIDEND/TRANSFER),symbol,shares,price,total,status}` |
| `performanceHistory[]` | Weekdays from 2025-12-04 to 2026-06-04, a random walk from 210000 with `(rand-0.47)*0.015` daily returns; the last point is forced to `account.totalValue` |
| `intradayData[]` | 5-minute points from 09:30 to 15:55, random walk; the last point equals the total value |
| `stockQuotes{}` | Built from holdings and watchlist, with randomized open/high/low/volume/avgVolume/marketCap/pe/eps/dividend/52-week values |
| `openOrders[]`, `faq[]` | 2 orders and 5 FAQs |
| `agentActivityLog[]` | Newest first, capped at 100 entries |
| `agentMetrics{}` | Per agent id: `{requests, avgLatency, successRate, tokensUsed, uptime, errors}` for portfolio-analyst, trade-assistant, market-research, customer-support, ken-agent |
| `logAgentActivity(entry)` | Scrubs PII from `entry.query` (the same 4 regexes as §12.4), then `unshift({timestamp: now ISO, ...entry, query})` and pops past 100 |
| `tickAllPrices()` | ±0.2% random tick on holdings and watchlist, ±0.1% on indices, then recomputes totals and weights (not currently scheduled) |
| `recalculatePortfolio()` | Recomputes every derived holding value, weights, `sectorAllocation` **in place** (keeping the object reference), and the account total, return, day change and percent. Used after a rebalance executes. |
| `getHolding(sym)`, `getQuote(sym)`, `getPortfolioSummary()` | Lookups. The summary returns totals, `holdingsCount`, top 5 holdings by value (note: sorts `holdings` in place) and a sector breakdown. |
| `formatCurrency(v)` | `Intl.NumberFormat('en-US',{style:'currency',currency:'USD'})` |
| `formatChange(v)` | Adds a `+` prefix when ≥ 0 to the currency format |
| `formatPercent(v)` | Adds a `+` prefix when ≥ 0, then `v.toFixed(2)+'%'` |
| `formatVolume(v)` | B/M (2 decimals) or K (1 decimal) |
| `getSectorForSymbol(sym,name)` | Static map for the 12 symbols, then keyword heuristics in this order: ETF-Tech (tech + ETF/FUND), ETF-Bonds, ETF-Commodity, ETF-Broad (ETF/INDEX/FUND/TRUST/… or SPY/VOO/IVV/VTI/GSLC/QEFA/EMGF), Technology (SOFTWARE/SEMICONDUCTOR/SYSTEMS/COMPUTERS), Healthcare, Financials, else `Other` |
| `getLiveMode()`, `getConnectionEnv()`, `getLiveAccounts()`, `checkEtradeStatus()`, `loadEtradeData()` | Live-mode plumbing (§8.3) |

Values that follow from the data, and that demo copy relies on: total holdings market value ≈ **$223,976**. **MSFT ≈ 15.9%** ($35,614.40) is the only holding above 15%. Technology ≈ **46.7%**. The day's driver (largest |dayChangeTotal|) is **NVDA** (+$434). The best mover is NVDA at +1.41% and the worst is TSLA at −1.16%. Verbatim constants are in **Appendix B**.

### 8.3 Live brokerage mode (optional for demo)
`checkEtradeStatus()` calls `/api/etrade/status`. When connected it sets `sessionStorage.etrade_connected = "true"` and runs `loadEtradeData()`:
- `GET /api/etrade/accounts` → `AccountListResponse.Accounts.Account[]`. For each account, in parallel, fetch balances, portfolio and transactions.
- **Balances:** take the first non-null of a long fallback chain over `BalanceResponse.Computed`, `accountBalance`, `cashAccountBalance`, `marginAccountBalance` (`netCash` → `cashBuyingPower` → `Cash.moneyMktBalance` → … for cash; `buyingPower` → … for buying power; `realTimeValues.totalAccountValue` → `netAccountValue` → … for total).
- **Positions:** `PortfolioResponse.AccountPortfolio[].Position[]` (handle the object-or-array shapes). Map to holdings: `Product.symbol`, `symbolDescription`, `quantity`, `pricePaid`, `currentPrice`, `daysGain|todayValuationChange`, `daysGainPct|todayValuationChangePct`, `marketValue`, `costBasis`, and sector via `getSectorForSymbol`.
- **Transactions:** `TransactionListResponse.Transaction[]` → `{date (postDate, date part), type, symbol, shares, price, total: amount, status:"Completed"}`.
- Compute per-account weights, sector allocation, totals, return and day change. Store in `liveAccounts`. Copy the first account into `account`, `holdings`, `transactions` and `sectorAllocation`, then call `render()`.

In `syncEtradeToMockData()`, live accounts become `mockData.accounts` entries with id `etrade_<accountIdKey>`, formatted strings, an `allocation.rows` from the sector allocation, and `movers` from the holdings. The original bank accounts are appended. The first name is taken from the first live account's name unless that word is MAIN, CORE or E\*TRADE.

Header link logic (`updateEtradeUI`): connected shows **"Log Off"** (calls `/api/etrade/disconnect`, clears sessionStorage, reloads). Disconnected shows **"Log On"**. Clicking it opens `/auth/etrade` in a 600×750 centered popup and changes the link to **"🔑 Enter PIN"** (amber gradient pill). The next click prompts for the PIN, calls `/auth/etrade/callback?pin=…`, alerts success and reloads, or alerts the failure and restores the label. The `.etrade-connect-prompt-btn` buttons share the same handler.

---

## 9. Router & navigation

### 9.1 Route table (`routeMeta`)
| Path | section (highlights primary nav) | secondary (highlights sub-nav) | view | title |
|---|---|---|---|---|
| `/`, `/accounts` | accounts | /accounts | dashboard | — |
| `/accounts/portfolios` | accounts | /accounts/portfolios | portfolios | — |
| `/accounts/watch-lists` | accounts | self | watch-lists | Watch Lists |
| `/accounts/orders` | accounts | self | orders | Orders |
| `/accounts/balances` | accounts | self | balances | Balances |
| `/accounts/activity` | accounts | self | activity | Activity |
| `/accounts/banking` | accounts | self | banking | Banking |
| `/accounts/tax-center` | accounts | self | tax-center | Tax Center |
| `/accounts/documents` | accounts | self | documents | Documents |
| `/accounts/dividend-reinvestment` | accounts | self | dividend-reinvestment | Dividend Reinvestment |
| `/accounts/open-account` | accounts | self | open-account | Open Account |
| `/pay-transfer` | pay-transfer | — | placeholder | Pay & Transfer |
| `/trading` | trading | — | placeholder | Trading |
| `/markets-ideas` | markets-ideas | — | placeholder | Markets & Ideas |
| `/at-work` | at-work | — | placeholder | At Work |
| `/planning` | planning | — | planning | Planning & Retirement |
| `/what-we-offer` | what-we-offer | — | placeholder | What We Offer |
| `/support` | support | — | placeholder | Support |
| `/alerts`, `/documents`, `/profile` | accounts | — | placeholder | Alerts / Documents / Profile |
| `/search` | accounts | — | search | Search |
| `/ai-insights` | ai-insights | — | ai-insights | AI Insights Hub |
| `/agent-studio` | geap-admin | — | agent-studio | Agent Studio |
| `/agent-fabric` | geap-admin | — | agent-fabric | Agent Fabric |
| `/ai-settings` | ai-settings | — | ai-settings | AI Settings |

Unknown paths fall back to the `/accounts` meta. All the "watch-lists/orders/…" views go to the `else` branch, which is `renderPlaceholder(meta.title)`.

### 9.2 Functions
- `getMeta(path = state.route)`.
- `routeTo(path, replace=false)`: parse with `new URL(path, origin)` and set `state.route = pathname`. For `/accounts/portfolios`, set `state.portfolioTab` from `?tab=` (default `allocation`). Call `pushState` or `replaceState` with the path, then `closeMenus()`, `closeMobileMenu()`, `render()` and `scrollTo(0,0)`.
- `setActiveNavigation()`:
  - Every `[data-section]` gets `aria-current="page"` when it matches `meta.section`; others have it removed.
  - Every `.secondary-nav a[data-route]` gets `aria-current` when its `href === (meta.secondary || state.route)`.
  - The Accounts sub-nav shows only when section is `accounts`. The **AI Admin sub-nav** (`#geap-secondary-nav`, holding Agent Studio | Agent Fabric) shows only when section is `geap-admin`. Both are hidden for other sections.
- `render()`: call `syncEtradeToMockData()` then `setActiveNavigation()`. If leaving agent-fabric, call `AgentFabric.cleanup()` to stop its interval. Then dispatch on `meta.view` (with the lazy loading from §5.3), call `bindViewEvents()`, and finally `app.focus({preventScroll:true})` inside `requestAnimationFrame`.
- **Global click delegation** on `document`:
  - `[data-route]` with an `href` starting with `/` triggers `preventDefault` and `routeTo(href)`. This also makes markdown links in chat route internally.
  - `[data-menu-button]` toggles the matching `.menu-popover[data-menu=name]`. It closes the others first, then positions the popover with `position:fixed` at `top = button.bottom + 8` and `left = max(12, button.left)`. A click outside any popover closes them all.
  - `.card-ai-analyzer-btn` sets `state.selectedAccountId = data-account-id`, works out a friendly name (`"<label> (<masked>)"`, or the Morgan Stanley account name for `morgan_stanley`), calls `render()`, then `triggerChatQuery("Run deep AI portfolio analysis for account: <name>", "portfolio-analyst")`.
  - The brokerage connect handler from §8.3.
- **Mobile menu:** `.mobile-menu-button` toggles `.primary-nav.is-open`, `aria-expanded` and `body.menu-open` (which sets `overflow:hidden`).
- **Search:** submitting `.search-form` (or pressing Enter in it) calls `routeTo('/search?q=' + encodeURIComponent(q))`, or `/search` when empty.
- **popstate:** set `state.route` from location, re-read `?tab`, and `render()`.
- `window.triggerChatQuery(message, agentId=null)`: un-hide `#agent-panel`, set the launcher's `aria-expanded="true"`, lazy-load chat (§5.3), then `Chat.init()`, `Chat.selectAgent(agentId)` when given, and `Chat.sendMessage(message)`.
- **Chat launcher click:** toggle `agentPanel.hidden` and `aria-expanded`. When opening, lazy-load chat and call `Chat.init()`.
- **Placeholder panel handlers** (only active before chat.js replaces the panel): `[data-close-agent]` hides it. `[data-agent-chip]` fills `#agent-input` with "Summarize my mock accounts." / "Triage my mock alerts." / "Explain the CORE mock allocation.". Submitting `.agent-panel__composer` appends `<p><strong>You:</strong> …</p><p><strong>GEAP placeholder:</strong> The next phase will answer this with route-aware mock tools and guardrails.</p>`.

### 9.3 `bindViewEvents()` (after every render)
- `[data-toggle-number]` toggles `state.visibleNumbers` and calls `render()`.
- `[data-toggle-account]` toggles `state.expandedAccounts` and calls `render()`.
- `[data-alert-tab]` sets `state.alertTab` and calls `render()`.
- `[data-allocation-mode]` sets `state.allocationMode` and calls `render()`.
- `#account-select` change sets `state.selectedAccountId` and calls `render()`.
- `[data-refresh]` sets the button text to `Refresh <refreshedAt>` (cosmetic).
- The AI briefing toggle `.markets-ai-card__toggle-btn` flips `state.aiDetailsExpanded` and runs `updateAIPanelUI()` **without a full re-render**. That function:
  - adds or removes `dashboard-hero--connected` on `.dashboard-hero` and `markets-ai-card--connected` on the card;
  - removes or adds `ai-details-section--collapsed` on `#ai-details-panel`;
  - rotates the chevron (0° when open, 180° when closed) and updates `aria-expanded`;
  - on open, if `InsightsEngine` is not loaded, shows an inline spinner ("Loading AI Insights...", an 18px ring with a `#6b2d9b` top border spinning at 0.8s) and lazy-loads insights CSS and JS, then injects `InsightsEngine.renderInsightsStrip()`. If it is loaded and `#insights-strip` is missing, it injects the strip.
- `.ai-details-card__close-btn` collapses the strip. `.markets-ai-card__details-trigger` expands it and smooth-scrolls to it. Both are optional hooks.
- `#include-external-toggle` change sets `state.includeExternalAccounts` and calls `render()`.
- `updateAIPanelUI()` also runs once after each render, to sync the panel state.

---

## 10. App shell (index.html)

`<html lang="en">`, title **"GEAP E\*TRADE POC"**, meta description "Internal GEAP proof-of-concept mock E\*TRADE workspace.", viewport meta.

Body structure, top to bottom:

1. `<a class="skip-link" href="#app">Skip to content</a>` (hidden above the viewport until focused).
2. `<header class="site-header">`:
   1. **Market strip** `<section class="market-strip" aria-label="Market snapshot">`, inner `.shell.market-strip__inner` (flex, gap 24, 0.72rem, horizontal scroll, nowrap):
      - `<strong>DOW</strong> 51,561.93` + badge `▲ 874.86 (1.73%)` (`.market-badge--positive`: bg `#d1e7dd`, green text)
      - `<strong>NASDAQ</strong> 26,830.96` + badge `▼ 23.02 (-0.09%)` (`--negative`: bg `#f8d7da`, red text)
      - `<strong>S&amp;P 500</strong> 7,584.31` + badge `▲ 30.63 (0.41%)`
      - `<span class="market-strip__timestamp">as of 10:23 AM ET</span>`
      - Right-aligned `.market-strip__actions`: `<a href="/agent-studio" data-route data-section="geap-admin" class="market-strip__admin"><span class="market-strip__admin-icon">⚙</span> AI Admin</a>`, `<span class="market-strip__separator">|</span>`, `<a href="#" class="market-strip__logoff" id="etrade-logon-link">Log On</a>`
   2. **Utility bar** `.utility-bar > .shell.utility-bar__inner` (flex, min-height 50):
      - Brand link `<a class="brand" href="/accounts" data-route aria-label="…">` containing an `<img class="brand__logo">` (38px tall, 216×57 intrinsic). The original inlines the brand wordmark as a base64 SVG data URI. **Use your own logo asset**: a black wordmark with a two-chevron mark in `#0F8EC7` and `#6633CC`.
      - `.mobile-menu-button` (3 bars + sr-only "Toggle navigation", `aria-controls="primary-nav"`), shown only at ≤920px.
      - `<nav class="utility-links" aria-label="Utility navigation">` (bold purple-bright links, gap 22):
        - `<button class="nav-button" data-menu-button="team">Morgan Stanley Team</button>` (with a caret drawn by `::after`)
        - `<button class="nav-button" data-menu-button="support">Support</button>`
        - `<a class="alert-link" href="/alerts" data-route><span class="alert-count">74</span> Alerts</a>` (count chip: bg `#eb2a00`, white)
        - `Documents` → `/documents`, `Profile` → `/profile`, `AI Settings` → `/ai-settings`
   3. **Primary nav** `<nav id="primary-nav" class="primary-nav">` (bg `--purple-dark` `#25085a`, height 38, white links 0.85rem, padding 0 14px). Links (each `data-route data-section`): Accounts, Pay & Transfer, Trading, Markets & Ideas, At Work, Planning, What We Offer, and `<a class="geap-nav-link"><span style="margin-right:5px">✦</span>AI Insights</a>`. The active link gets a white 8px CSS triangle at bottom center. On the far right is the search form (`role="search"`, max-width 180, a 30px pill input with translucent white fill and border, white placeholder "Search", and a magnifier drawn with `::before` (ring) and `::after` (handle)).
   4. **Accounts sub-nav** `.secondary-nav` (white, 32px, 0.82rem, horizontal scroll): Complete View, Portfolios, Watch Lists, Orders, Balances, Activity, `<button class="secondary-dropdown" data-menu-button="banking">Banking</button>`, Tax Center, `<button … data-menu-button="documents">Documents</button>`, Dividend Reinvestment, Open Account. The active item has a 4px `--purple-bright` underline inset 14px on each side.
   5. **AI Admin sub-nav** `<nav class="secondary-nav geap-secondary-nav" id="geap-secondary-nav" hidden>`: Agent Studio, Agent Fabric.
3. **Menu popovers** (siblings of the header, `hidden`, `.menu-popover`, fixed-positioned by JS):
   - team: Advisor contact → /accounts, Schedule review → /planning, Shared documents → /documents
   - support: Support center → /support, Message center → /alerts, Forms and documents → /documents
   - banking: Banking summary → /accounts/banking, Transfer money → /pay-transfer, Balances → /accounts/balances
   - documents: Statements → /accounts/documents, Tax documents → /documents, Tax center → /accounts/tax-center
4. `<main id="app" tabindex="-1">` with the static skeleton from §5.4.
5. `<button class="chat-launcher" aria-expanded="false" aria-controls="agent-panel"><span class="chat-launcher__icon" aria-hidden="true"></span> Chat</button>`: fixed bottom 24 / right 28, pill, `--purple-dark`, weight 900, min-height 48, padding 0 22, shadow. The icon is a small speech bubble drawn in CSS.
6. `<aside id="agent-panel" class="agent-panel" aria-label="GEAP assistant panel" hidden>`: fixed bottom 84 / right 28, `width:min(390px, 100vw-32px)`, `max-height:min(560px, 100vh-120px)`, white, flex column, shadow, z-index 25. Placeholder content: header "GEAP Assistant / Mock data only" with an `x` close button, a body paragraph and 3 chips, and a composer (`#agent-input`, Send). `Chat.init()` replaces all of it (§12.1).
7. The eager scripts (§5.2).

---

## 11. Views

Common building blocks (all in `styles.css`):
- **`.page-band`**: a white full-width band around `.shell`. `.page-band--gray` uses the `--band` `#f5f5f5` background.
- **`.page-title-row`**: flex, space-between, holding `h1` and a toolbar.
- **`.outline-button`**, **`.text-button`**, **`.link-button`**, **`.icon-button`**, **`.button`**: see the CSS appendix.
- **`.switch`**: a label containing a checkbox and a `<span aria-hidden>` drawn as a track and knob. Checked state turns purple.
- **`.geap-card`**: used with heavy inline styles. White, `1px solid #e2e8f0`, radius 8, padding 16–24. Some cards carry a **4px gradient top bar** (`linear-gradient(90deg,#6B2D9B,#4F8EF7)`, absolutely positioned).
- **`.geap-badge`** (pill), with variants `--green`, `--blue`, `--red`, `--amber` and `--purple`. **`.geap-badge-dot`** (8px circle), with `--active` (green, pulsing glow), `--warning` and `--error`.
- **Disclosures footer**: `renderDisclosures()` is appended to the dashboard, placeholder and portfolios pages (§11.9).

### 11.1 Dashboard (`renderDashboard()`)

```
┌ section.dashboard-hero (white) ─────────────────────────────────────────────┐
│ .shell > .dashboard-controls  (grid 1fr auto 1fr, gap 28; 1 col ≤1220px)    │
│  [welcome]                 [markets-ai-card]                [dashboard-actions] │
└─────────────────────────────────────────────────────────────────────────────┘
section#ai-details-panel.ai-details-section(--collapsed)  → InsightsEngine strip
section.page-band.page-band--gray > .shell.content-grid (1fr | 340px)
   .account-stack:  account cards…, Morgan Stanley Accounts (opt), Watch Lists Snapshot
   aside.side-rail: Holdings Overview, Allocation by Sector, Recent Activity,
                    Morgan Stanley Team, Core Portfolios Support Team, Launch Pad, Alerts
footer.footer-disclosures
```

**Welcome column** (`.welcome`):
- `h1` "Welcome, Ken"; `p` "Last login: Jun 04, 2026, 8:07 PM ET" (0.65rem, muted).
- `.metric-row` with two `.metric` blocks:
  - "Total Assets" / **$48,912.74** / link `View details →` (`.metric__link` → /accounts/portfolios)
  - "Day's Gain" / `.metric__value-row > .metric__value-text`: `<strong class="positive">+$318.22</strong>` and `<span class="metric__pct positive">(0.65%)</span>` (hard-coded), plus link `View performance →`
- Skeleton variant: inline skeleton bars instead of the values.

**GEAP Agent Briefing card** (`.markets-ai-card`, max-width 540, centered, radius 12, padding 14×18, subtle shadow; the `--connected` modifier changes its look when the strip is open):
- Header: `✦` (`#6B2D9B`) + **"GEAP Agent Briefing"** (700, 0.95rem, `#1a1a2e`) + "Since Jun 8" (muted 0.78rem). On the right, `.markets-ai-card__toggle-btn` (aria-label "Toggle AI Details") with a 12px chevron SVG (`M2 8L6 4L10 8`, rotated 180° when collapsed).
- Content: a 4-column grid (gap 16). Each column is a flex column with a right border `#cbd5e1` except the last:
  1. **$132** (16px, 800, `#137a27`) / "Harvestable Losses" (12px, 700) / "Tax Specialist" (10px muted)
  2. **15.9%** (`#d97706`) / "MSFT Exposure" / "Risk Watchdog"
  3. **12** (`#0669d8`) / "Diagnostics Passed" / "System Health"
  4. "Portfolio up" or "Portfolio down" (by the sign of `BrokerageData.account.dayChange`) / `"+0.50% (+$318.22)"` (format: `formatPercent(account.dayChangePercent)` + ` (` + `mockData.totals.dayGain` + `)`) / "Led by **NVDA**" (the holding with max |dayChangeTotal|, in blue)
- Skeleton variant: four skeleton columns.

**Dashboard actions** (`.dashboard-actions`, right column):
- `button.text-button[data-refresh]` "Refresh `<span>`Jun 04, 2026, 10:19 PM ET`</span>`"
- `button.outline-button` "Add external accounts"
- `label.switch` "Include external accounts" with `<input type=checkbox id="include-external-toggle">`
- `.dashboard-actions__row` (right-aligned): `⚙ Customize` | `Learn More <span class="insights-info-icon">ⓘ</span>`

**AI details strip** (`#ai-details-panel.ai-details-section`): collapsed by default via `--collapsed` (CSS animates max-height and opacity). The inner `.ai-details-inner` holds `InsightsEngine.renderInsightsStrip()` (§11.4.1) when available.

**Account cards** (`renderAccountCard(account)` for each non-bank account; the checking account is rendered with the bank template only in live mode because the dashboard filters `!account.bank`):
- `<article class="account-card" data-account-card="id">`, then `.account-card__top` (grid: info | metrics; 1 col ≤1220px):
  - Info: optional `<p>subtitle</p>`. Then `h2.account-title` containing `<strong>label</strong> <em>-1864</em>` (the masked number, or the full number when revealed) and `button.link-button[data-toggle-number]` labeled "Show number"/"Hide number". When `account.beneficiary` is set, it is followed by `.beneficiary`: an `i` badge (`.notice__icon`) and "You don't have a beneficiary for your account." with links "Add beneficiary" (→/profile) | "Set reminder".
  - Metrics `.account-metrics`: a `.metric-line` per row (label span, value strong with tone class). The rows are "Net Account Value", "Day's Gain" (toned), "Available for Withdrawal" and "Cash Purchasing Power". For the **stock** account they are "Current Account Value", "Day's Gain" and "Potential Benefit Value" (3 rows). Underneath is `button.text-button[data-toggle-account]` labeled "Show more"/"Show less".
- `.account-card__links`: `[data-toggle-account]` "`>` Portfolio snapshot" (`v` when expanded; not shown for stock) `|` "Open orders (0)" (→/accounts/orders) `|` "Quick links ..." `|` **"✦ AI Analyzer"** (`.link-button.card-ai-analyzer-btn[data-account-id]`, purple, weight 600).
- When expanded and the account has movers, a **movers table** follows: `.mini-tabs[role=tablist]` with "Top Movers (n)" (selected) and "Portfolio News". Then `.table-wrap > table` with columns Symbol (link to /trading) | (Trade outline button) | Change % | Last Price $ | Change $ | Day's Gain $, where the % and $ cells are toned. Then `.table-footer`: "n Total | View full portfolio" and "Market Closed Jun 04, 2026, 4:00 PM ET".
- Default: `elcie` is expanded.
- The bank template (`renderBankAccountBody`) has the subtitle and title, the metrics "Available Balance" and "Total Balance", and the links "Transfer money" | "Quick links ..." | "✦ AI Analyzer" | "FDIC Insured - Backed by the full faith and credit of the U.S. Government".
- Disconnected (skeleton) variant: two skeleton account cards. Data shown but zero accounts: the "E\*TRADE Connection Required" empty state.

**Morgan Stanley Accounts** (`renderMorganStanleyAccounts()`, only when `includeExternalAccounts`): `section.wide-panel` with `h2` "Morgan Stanley Accounts" and a table with columns Account | Assets $ | Day's Gain $ | Day's Gain %, holding one row (Premier CashPlus - 7306, $8,104.31, $0.00, 0.00%). The footer has "Go To Morgan Stanley Online | Quick links ... | ✦ AI Analyzer (data-account-id=morgan_stanley)" and "Last updated as of <refreshedAt>". Toggling it on also adds $8,104.31 to Total Assets, giving **$57,017.05**.

**Watch Lists Snapshot** (`section.wide-panel`): `h2` "v Watch Lists Snapshot". A `select#watch-list-select` with Investments to Watch / Core Portfolio Watch / Income Ideas. A table with columns Symbol | Trade | Last Price $ | Change % | Change $ | Volume holding RIVN and MBLY. Footer: "2 total | View full watch list" and "Market Closed Jun 04, 2026, 4:00 PM ET".

**Side rail** (`aside.side-rail`, grid gap 24). Each card is a `.portfolio-summary-card` (white, bordered, rounded) with `h3`, `.portfolio-summary-card__sub` (muted) and `.portfolio-summary-card__link` (purple arrow link):
1. **Holdings Overview**, "See your top holdings and allocation.". `.holdings-overview` holds a 90×90 SVG donut (viewBox 100): a white r25 base, sector slices as `M50,50 L… A35,35 …` paths using the palette `['#6B2D9B','#3b82f6','#22c55e','#f59e0b','#ef4444','#8b5cf6','#06b6d4','#ec4899']` at opacity 0.85 (a sector ≥99.9% is drawn as a full circle), and a white r22 hole. Beside it, `.holdings-list` with header "Top 3 holdings" and the top 3 by market value (MSFT, VTI, NVDA in demo), each with a 14px brand glyph SVG for MSFT, NVDA or AAPL (others get `●`), the name and `weight.toFixed(1)%`. Link: "View full portfolio →". Empty: "No active holdings to display."
2. **Allocation by Sector**, "See how your portfolio is allocated.". The top 4 sectors plus "Other" (the sum of the rest) as `.sector-bar-row` items (name | track with fill `#6B2D9B` at weight% | pct with 0 decimals). Link: "View allocation details →".
3. **Recent Activity**, "View your latest account activity.". The first 3 transactions as `.recent-activity-row` (description, falling back to "Buy NVDA" style text built from type and symbol | date | amount with a +/− sign, green for DIVIDEND/DEPOSIT and red otherwise, with 2 decimals). Link: "View all activity →" (→/accounts/activity).
4. **Morgan Stanley Team**: `.team-summary` with "Financial Advisor", **VIRTUAL ADVISOR**, `tel:888-454-0555` link, and a round avatar with a users icon SVG in `#6B2D9B`. Link: "View all team members →".
5. **Core Portfolios Support Team**, "Dedicated customer support specialist line.", a phone icon and **866-484-3658** (tel link, `#6B2D9B`, 1.1rem bold).
6. **Launch Pad**, "Quick access to Wealth Management tools.". `.side-list` with 5 rows: title bold, detail muted, and a small outline button (purple border, 11px, radius 4) labeled Launch or Sign up.
7. **Alerts (74)**, with a "Message Center" link on the right, "See your latest notifications.". `.alert-tabs[role=tablist]` with All / Account / Symbol (`aria-selected`, underline on the selected one) filtering `mockData.sidePanels.alerts` by category. `.alert-item` rows show the title (0.82rem bold) and date (0.72rem muted). Footer links: "View all →" | "Set and manage alerts →".

Each side-rail card has a matching skeleton variant.

### 11.2 Portfolios (`portfolios.js`, `renderPortfolios()`)

- **Skeleton** (not showData): header with skeleton tabs and summary, a 300px skeleton card, and disclosures.
- **No account** (`selectedAccount()` undefined): header plus the "E\*TRADE Connection Required" empty state (the copy mentions portfolio holdings).
- `selectedAccount()` returns the account matching `state.selectedAccountId`, or else `mockData.accounts[1]`.
- `getSelectedAccountHoldings(account)` returns `account.holdings` for `etrade_*` accounts and `BrokerageData.holdings` otherwise. **Every mock account shows the same 12 BrokerageData holdings.**
- Computations: `totalEquitiesVal = Σ marketValue`; `cashVal` is the live account's cash or `BrokerageData.account.cashBalance`; `totalVal = equities + cash`; `equitiesPct` and `cashPct` are formatted with 1 decimal.

Layout:
1. `section.portfolio-header > .shell`:
   - `.page-title-row`: `h1` "Portfolios" and a `.toolbar` with Refresh `<refreshedAt>`, Help (→/support), and icon buttons "P" (Print) and "D" (Download).
   - `nav.portfolio-tabs[aria-label="Portfolio tabs"]` with 7 anchors whose `onclick` does `state.portfolioTab='…'; render(); return false;`. The active one has `aria-current="page"`: **Positions, Allocation (default), Performance, Historical Value, Gains & Losses, Risk Assessment, Estimated Income**.
2. `section.portfolio-summary > .shell.portfolio-summary__grid`:
   - `.field`: `label` "Account" + `select#account-select` listing every non-bank account as `"<label> <masked>"`, then `<p>` with the subtitle or type.
   - `.summary-metric`: Net Account Value (positive) / Tax Loss Harvesting `$0.00` / link "Unenroll"
   - `.summary-metric`: Total Unrealized Gain (positive) / My Portfolio `<portfolioName>` / link "Investment Proposal"
   - `.summary-metric`: Day's Gain Unrealized (toned)
   - `.summary-action`: outline button "Move Money"
3. Unless the tab is risk, a `.notice` row: an `i` icon and **"Register now:"** "Core Portfolios Quarterly Commentary featuring Morgan Stanley's Global Investment Office" webinar.`<br>`Join this webinar live or on demand to hear quarterly market commentary.
4. Tab content:
   - **allocation**, plus performance and income (which fall through to the same content): `section.allocation-card` with `h2` "Allocation" and `.mini-tabs` "Asset class"/"Sector" (`[data-allocation-mode]`).
     - **Asset mode:** `.allocation-body` (2 cols, 1 col ≤1220px). A `.donut` (CSS circle, max 310px, `conic-gradient(#6B2D9B 0% X%, #22c55e X% 100%)` with a white center from `::after`) containing `.donut__label` `<strong>$total</strong><span>Market value</span>`. Next to it, a table with columns Asset class | Portfolio % | Market value $ and the rows **Total market value 100.0%**, Equities, Cash.
     - **Sector mode:** uses `account.allocation.rows` (for core: Equities 98.1% / Cash 1.9%). The conic gradient is built from the cumulative pct using the 8-color palette, defaulting to `#1177aa` when empty. The table has columns Sector | Portfolio % | Market value $, a total row, and the empty text "No sector breakdown available.".
     - Then `section.holdings-sections` with two `.holding-section` tables ("Equities" / "Cash"), columns Description | Symbol | Qty | Price $ | Target % | Portfolio % | Market value $, and one total row each ("> Total equities", target 100.0%, …; "> Total cash", target 0.0%).
   - **positions:** `section.positions-card` (panel, padding 24, radius 6) with `h2` "Positions (n)" and a table with columns Symbol (bold link) | Description | Qty | Price $ | Avg Cost $ | Portfolio % (2 decimals) | Day's Gain $ (`formatChange(dayChangeTotal) (formatPercent(dayChangePct))`, toned) | Market Value $ (bold). Empty: "No positions found in this portfolio."
   - **gains:** `h2` "Gains & Losses" and columns Symbol | Description | Qty | Avg Cost $ | Price $ | Total Cost $ | Current Value $ | Total Gain $ (`formatChange(totalReturn) (formatPercent(totalReturnPct))`, toned).
   - **historical:** `h2` "Historical Performance" and a Date | Account Value table showing the last 15 points of `performanceHistory`, newest first.
   - **risk** (`renderRiskAssessment(account, holdings)`): a `.content-grid` with two columns.
     - **Left:**
       - Hero `.geap-card` with a gradient top bar: **"✦ AI Portfolio Risk Diagnostic"** (18px 700) / "GEAP Real-Time Asset Analysis", a pill badge "Active Monitoring" (`#eef7fc` bg, `#4F8EF7` text, `#bce0fd` border), the paragraph *"The Google Enterprise Agent Platform monitors your active account portfolio against standard regulatory and custom firm compliance policies. Our diagnostics have detected sector concentration and single-stock exposure thresholds that may exceed standard risk parameters."*, and a purple filled button **"✦ Run Active Risk Analysis"** that calls `triggerChatQuery('Run a comprehensive risk analysis on my portfolios and active account holdings.')`.
       - `.risk-factor-grid` (2×2) of cards: an uppercase 12px label, a status badge, a 24px value and a 12px caption.
         1. **Single-Stock Exposure**: max holding weight within equities. >15 → "High Risk" (red badge, "`SYM` position exceeds 15% single-holding limit."); >10 → "Elevated" (amber); else "Low Risk" (green). No equities → "No active equity holdings to analyze."
         2. **Portfolio Beta**: Σ(marketValue/totalVal × sectorBeta) with sector betas Technology 1.35, Consumer Disc. 1.2, Financials 1.1, ETF-Broad 1.0, ETF-Tech 1.35, ETF-Bonds 0.15, ETF-Commodity 0.2, Healthcare 0.7, Other 1.0. >1.25 → "Elevated" / "Highly reactive to market volatility."; <0.50 → "Low Volatility" / "Conservative posture with lower volatility."; else "Moderate" (blue badge) / "Aligned with broad market volatility.".
         3. **Sector Exposure**: max sector % including Cash as a sector. >30 → High Risk "`X` sector concentration exceeds benchmark."; >20 → Elevated "… exposure is elevated but balanced."; else Low Risk "… within balanced limits.".
         4. **Estate Planning**: complete when `account.beneficiary === true` or the label contains UTMA/UGMA/Custodial/Trust/IRA, giving "Designated" / "Complete" (green) / "Beneficiary designation active for this account."; else "None" / "Incomplete" (amber) / "Missing beneficiary designation for this account.".
       - Badge styles: green `#f0fdf4`/`#15803d`/`#bbf7d0`, red `#fef2f2`/`#ef4444`/`#fecaca`, amber `#fffbeb`/`#d97706`/`#fef3c7`.
     - **Right:** a full-height card **"ACTIVE AI SAFEGUARDS"** with these rows: Safety Filter ● Active, PII Redaction ● Active, Financial Disclaimers ● Active, Audit Compliance Logging ● Active (green `#16a34a`), Human Escalation Router ● Ready (slate). A gray note box: "All advisor interactions are audited and logged for compliance monitoring. Visit the **Agent Fabric** to inspect active routing histories." (the link routes).
5. `renderDisclosures()`.

Deep link: `/accounts/portfolios?tab=risk` opens the Risk tab. The insight cards link there.

### 11.3 Planning & Retirement (`planning.js`)
- A `.page-band` title row: "Planning & Retirement" + Refresh.
- `section.shell` (margin 24/40) holding a `.content-grid`:
  - **Left:**
    - Hero `.geap-card` with a gradient bar: **"✦ GEAP Intelligent Retirement Advisor"** / "Active Wealth & Retirement Planning", a pill badge "AI Enabled" (pink `#fdf2f8`/`#db2777`/`#fbcfe8`), and the paragraph *"The Google Enterprise Agent Platform retirement optimizer simulates target date glide paths, runs Monte Carlo projection maps, and assesses your asset allocation to ensure a secure path to retirement. Click below to trigger a retirement health check directly with the assistant."* A purple button "✦ Run Retirement Health Check" calls `triggerChatQuery('Run retirement planning health check.')`.
    - `.planning-stats-grid` (4 cards: uppercase 11px label, 22px value): Retirement Target Age **65** (in 2046) · Retirement Savings **$84,230.15** · Projected Value (Age 65) **$1,245,000** · Probability of Success **84%** (High), shown in green.
    - `.planning-tools-grid` (3 cards): "📈 Glide Path Optimizer" / "Maintains your equity-to-bond ratio as you age to automatically protect your capital assets."; "🎲 Monte Carlo Projections" / "Runs 1,000 distinct market permutations to evaluate potential drawdown risks and asset longevity."; "💼 Asset Allocation Review" / "Reviews cash, domestic, and international equity balances to preserve long-term compounding."
  - **Right:** card "Active AI Safeguards" with these rows (title + 10px subtitle + green "● Active"): PII Protection / SSN & account scrub active; Retirement Safety Rules / Fiduciary advice limits active; Investment Disclaimer / SIPC warning disclosures active. Then the same Agent Fabric note box ("advisory interactions").
- `.planning-stats-grid` and `.planning-tools-grid` are defined in styles.css (responsive grids).

### 11.4 AI Insights

#### 11.4.1 `InsightsEngine` (dashboard strip)
`generateInsights()` builds these from BrokerageData:
1. **briefing** (always): icon 🌅, title "Morning Briefing". Body: "Your portfolio is **up $X** (+P%) today, led by **SYM** (+p%). You're **outperforming** the S&P 500 by Npp today." (or "S&P 500 is ahead by Npp today."). Here X = |account.dayChange|, the driver is the max |dayChangeTotal|, and outperformance = dayPct − SPX.changePct. Action: "View Portfolio" → /accounts/portfolios.
2. **risk** (if any holding weight > 15): ⚠️ "Concentration Alert", "**MSFT** is 15.9% of your portfolio ($35,614.4). Single-stock positions above 15% increase risk." Action: "Rebalance Ideas" → `/accounts/portfolios?tab=risk`.
3. **diversification** (if the top sector is > 40%): 📊 "Sector Concentration", "Your portfolio is **47% Technology**. Broad market indices typically cap sector exposure at ~25%. Consider adding exposure to ETF - Bonds or Financials." Action: "View Allocation". This card also renders a 90×90 **mini pie** SVG (r40 slices, palette `['#6B2D9B','#22c55e','#3b82f6','#f59e0b','#ef4444','#8b5cf6','#06b6d4']`).
4. **mover** (if the best dayChangePct > 0.5): 🚀 "Top Mover", "**NVDA** is up **1.41%** today (+$434.00). Meanwhile, TSLA is down -1.16%." It carries the best and worst symbol and pct.

`renderInsightsStrip()` produces `div.geap-insights-strip#insights-strip`:
- `.insights-cards-row` (flex):
  - `.insights-cards-main` (3-col grid): up to 3 non-mover cards as `.insight-card-v2.animate-fade-in` (animation-delay `i*100ms`). Each holds `.insight-card-v2__inner` (white): a header with `__icon-new` and `__title-new`, a body with `__text` and an optional `__pie`, and an optional `a.insight-card-v2__action[data-route]` reading "Label →".
  - `.insights-actions-col` (140px column): `a.insights-view-all-overlay[href=/ai-insights]` "✦ View all insights", and `button.insights-run-diag-btn` "Run Diagnostic Triage →", which calls `triggerChatQuery('Triage my active alerts')`.
- If there is a mover, `.insights-bottom-row > .insight-mover-card` (delay 300ms) contains: a left icon; info (title + text); `.insight-mover-card__tickers` with `.mover-ticker--up` (symbol, `+x%` green, a 65×28 green polyline sparkline) and `.mover-ticker--down` (red); a `|` separator; and a rebalance block "💡 **Rebalance Ideas** / Improve diversification and reduce concentration risk. / View suggestions →" (→ risk tab).
- Skeleton variant: 3 skeleton cards and a skeleton actions column.
- Helper graphics `morningBriefingGraphic()`, `concentrationAlertGraphic()` and `decorativeWave()` exist (decorative SVGs) but are unused.

#### 11.4.2 `AIInsightsPage` (`/ai-insights`)
- **Header band** (`.geap-page-header`): `h1` "✦ AI Insights Hub" (the star in `#6B2D9B`), subtitle "Personalized wealth intelligence powered by Google Enterprise Agent Platform", and on the right a badge `geap-badge--green` with an active dot and "GEAP Active".
- **"Your Dedicated AI Advisory Fleet"**: `.agent-team-grid` (4 cols) of `.agent-card`. Each has a header (emoji icon + `.agent-card__status` with a pulsing dot and "Active"), `h3.agent-card__name`, `p.agent-card__desc` and `.agent-card__footer`:
  - 📈 Portfolio Optimizer: "Monitors asset weights, sector tilts, and triggers alerts for rebalancing opportunities." / "Last Action: Scanned portfolio just now"
  - 📝 Tax Specialist: "Tracks tax-loss harvesting targets and reviews capital gain scenarios across positions." / "Last Action: Scanned holdings today, 8:00 AM"
  - 🌅 Retirement Guide: "Simulates wealth glide paths, runs Monte Carlo projection maps, and assesses longevity goals." / "Last Action: Recalculated milestones today, 9:15 AM"
  - 🛡️ Risk Watchdog: "Enforces regulatory constraints, safety guardrails, and compliance disclaimers on all advice." / "Last Action: Audited session logs just now"
- `.ai-insights-dashboard-grid` (2 cols, margin-top 32):
  - **Left: "Proactive Agent Recommendations"**, three `.insight-feed-card.animate-fade-in` (delays 0/100/200ms). Each has a header (`.insight-badge.{risk|progress|tax}` + the source agent name), an `h3`, a `p` and a `button.insight-feed-btn`:
    1. Risk Alert · Portfolio Optimizer · "Portfolio Concentration" · "Your MSFT position represents **15.9%** of your total assets. Diversification guidelines recommend capping individual equities at 15.0% to minimize single-stock downside and sector overlap." · "✦ Run Rebalance Simulation" → `triggerRebalance()`
    2. Progress Update · Retirement Guide · "Retirement Wealth Tracking" · "Based on your current assets ($84,230.15) and saving rate ($1,200/mo), the Monte Carlo model projects an **84%** success rate of reaching your $1.0M target by age 65 (2046)." · "✦ Run Retirement Health Check" → `triggerRetirement()`
    3. Tax Opportunity · Tax Specialist · "Tax-Loss Harvesting Alert" · "We have identified **$450** in harvestable capital losses across your active holdings that could offset future capital gains taxes. Click to run harvesting analysis." · "✦ Analyze Tax Harvesting" → `triggerTaxHarvest()`
  - **Right: "What-If Scenario Sandbox"** (`.sandbox-card`). Intro: "Simulate transactional outcomes before placing actual trades. GEAP agents will classify intent, calculate sector shift, and audit compliance guardrails.". `.sandbox-form` contains Action (`select#sandbox-action` Buy/Sell), Symbol (`select#sandbox-symbol` NVDA (NVIDIA) / MSFT (Microsoft) / TSLA (Tesla) / AAPL (Apple)) and Quantity (`input#sandbox-qty` number, min 1, default 100). Then `button.sandbox-action-btn#run-sandbox-btn` "✦ Analyze Trade Scenario" and `#sandbox-result-area`.
- Sandbox state is module-level: `{symbol:'NVDA', action:'Buy', quantity:'100', analyzing:false, analyzed:false, step:0}`. `bindEvents()` syncs the select and input values into it.
- `runAnalysis()` reads the form, sets analyzing, then advances steps 1..4 at 250ms intervals and finishes at 1250ms with `analyzed=true`. Each step re-renders only `#sandbox-result-area`.
- Result states:
  - **Idle:** a dashed box reading "Specify a mock trade transaction above and click analyze to launch the GEAP simulation."
  - **Analyzing:** a gray box with a spinner and "GEAP Orchestrating Multi-Agent Evaluation..." above a `.sandbox-stepper` of 4 `.stepper-item` (`completed`/`pending`, a dot and text): "Orchestrator classifying trade intent...", "Portfolio Analyst computing sector weight shift...", "Risk Watchdog assessing portfolio Beta delta...", "Safety Engine checking compliance safeguards...".
  - **Analyzed.** The math:
    - prices NVDA 120, MSFT 380, TSLA 220, AAPL 180; `totalCost = qty*price`.
    - `cash` = `parseFloat(selectedAccount().available)` (default 1000).
    - Starting weights: NVDA 5.2, MSFT 72.8, TSLA 4.1, AAPL 6.4.
    - `totalVal` = the selected account's netValue (default 12734.88); `stockStart = weight% × totalVal`.
    - **Buy:** `stockEnd += cost`, `newTotal += max(0, cost − cash)`, and it is out of bounds if cost > cash ("Insufficient Funds: The simulated purchase of $X exceeds your available buying power ($Y).").
    - **Sell:** `owned = round(stockStart/price)`, and it is out of bounds if qty > owned ("Insufficient Holdings: The simulated sale of N shares of SYM exceeds your current holdings of M shares."). Otherwise subtract the sale value.
    - `endWeight = stockEnd/newTotal × 100`.
    - Beta starts at 1.42 and ends at (Buy/Sell): NVDA 1.54/1.34, MSFT 1.45/1.39, TSLA 1.59/1.31, AAPL 1.44/1.40. For qty > 1000, scale the delta by `min(2.5, log10(qty) − 1)`.
    - Direction color: red for Buy, green for Sell.
  - **Out of bounds:** a red panel (`#fdf2f2`/`#fca5a5`, fade-in) titled "⚠️ Guardrail Alert: Out of Bounds". `.sandbox-results-grid` shows "Simulated Cost" and "Projected Weight", then the warning in bold, then (if endWeight > 30) "**Concentration Risk:** This transaction would also create severe concentration risk, putting SYM at **X%** of your total assets, exceeding the standard 15% safety threshold.". A red CTA "✦ Ask Assistant About This Simulation".
  - **OK:** a green panel (`#f0fdf4`/`#bbf7d0`) titled "✔ Simulated Outcome Analysis", with metrics "SYM Portfolio Weight: a% ➔ b%" and "Estimated Beta Delta: 1.42 ➔ x (±d)" (colored), the text "This buy transaction increases tech sector dependency. Risk engines report compliance constraints are **fully satisfied**." (decreases for sell), and the CTA.
- The trigger functions each ensure chat is loaded, then call `Chat.init()`, `Chat.selectAgent('portfolio-analyst')`, `Chat.sendMessage(<text>)` and open the panel:
  - triggerRebalance: "Run a comprehensive risk analysis on my portfolios and active account holdings"
  - triggerRetirement: "Run retirement planning health check."
  - triggerTaxHarvest: "Show tax loss harvesting opportunities and run tax analysis."
  - askAssistant: "Analyze my simulated transaction of buying|selling <qty> shares of <SYM>."

### 11.5 AI Settings (`settings.js`, `/ai-settings`)
- Title band: `h1` "AI Settings & Guardrails" (24px 700) / "Manage security guardrails, trade permissions, and information access rights.". On the right, `#settings-status-indicator` (opacity 0 → 1 on save, text "✓ Guardrails auto-saved at HH:MM:SS") and Refresh.
- `.settings-grid` (2 cols):
  - **Left card** (gradient top bar) "🔒 System Access & Guardrails":
    - `.settings-group` "👤 System Access Level" holding `.radio-card` labels (the `active` class marks the checked one), each with a hidden-ish radio and `.radio-card-content` (`-title`, `-desc`):
      - `read-only` "Read-Only Access": "AI can only inspect your portfolio, balances, and account summaries. All UI page routing, rebalancing tools, and write parameters are fully blocked."
      - `read-write` "Read-Write Access (Recommended)": "AI can navigate the platform UI, select account views, switch charts, and stage rebalance tickets on your behalf."
    - "⚡ Trade Execution Guardrails":
      - `execute` "Execute Trades Directly": "AI can execute rebalance trades directly after your confirmation click in the interactive chat ticket. Holdings and cash are modified instantly."
      - `draft` "Draft / Stage Trades Only": "AI can only prepare trade tickets/widgets. Clicking execution stages the order in your manual review queue; no holdings are updated directly."
      - `never` "Never Execute Trades": "Blocks all trade actions, rebalancing, and buying/selling commands. The AI cannot stage or execute any trading operations."
    - "🛡️ Privacy & Security": a `.switch-row` with "Active PII Scrubbing" / "Automatically redact personally identifiable data (SSNs, accounts, emails, and phone numbers) before sending chat query payloads." and a switch `#settings-pii-scrub`.
  - **Right card** "🌐 Info Source Management" / "Whitelist or blacklist specific public research libraries and private account documents accessible by the AI.":
    - "PUBLIC SOURCES (MARKET RESEARCH)": `.source-checkbox-item` rows (emoji logo + name | `.source-checkbox-badge.public` "Public" + a switch `input[data-source]`): 📰 Yahoo Finance (`yahoo`), 🏛️ SEC Edgar (`sec`), 💼 Bloomberg (`bloomberg`), 🌐 Reuters (`reuters`), ⭐ Morningstar (`morningstar`).
    - "PRIVATE SOURCES (ACCOUNT ACCESS)" (badge `.private` "Private"): 📊 Portfolio Holdings (`private_holdings`), 📜 Transaction History (`private_transactions`), 📝 Tax Records & Statements (`private_tax`).
- A toast `#settings-save-toast.settings-save-toast` "✓ AI settings auto-saved successfully". Adding `.show` slides it in; it is removed after 2.5s via `window.toastTimeout`.
- `bindAISettingsEvents()`:
  - Clicking a `.radio-card` (but not its input) checks the radio and dispatches `change`.
  - A delegated `change` handler on `#app` updates `state.aiSettings` (accessMode, tradeExecutionMode, piiScrubbingEnabled, `allowedSources[data-source]`), syncs `.active` on the radio cards, writes `localStorage.geap_ai_settings` and shows the indicator and toast.
  - **Fix in the rebuild:** the delegated listener is attached to `#app` on every visit, so it stacks. Attach it once, or use an AbortController.

### 11.6 Agent Studio (`agent-studio.js`, `/agent-studio`)
- Header: "Agent Studio" / "Design, configure, and test AI agents", badge "● GEAP Connected".
- `.studio-layout`, a 3-column grid (sidebar | canvas | config):
  - **Sidebar `.studio-sidebar`:**
    - Header "AGENT FLEET".
    - `.studio-agent-list` of `.studio-agent-item[data-select-agent]` rows (icon 20px, name 13px 600, description 11px tertiary, a green dot on the right; the active one is highlighted).
    - A full-width outline button "+ Create New Agent".
  - **Canvas `.studio-canvas`:**
    - Header with "`icon` `name`" (18px 600), an "Active" badge and the outline button "💾 Save Changes".
    - A **flow diagram**, centered vertically: `.flow-node` "👤 User Query / Natural language input" (light blue gradient) → `.flow-connector` → `.flow-node.router` "🧠 Agent Router / Intent classification" → connector → `.flow-branches` holding one clickable `.flow-node[data-select-agent]` per agent (icon + name, min-width 120, `active` on the selected one).
    - `.studio-test-console`: header "🧪 Test Console" + a "Clear" link-button; `#test-console-body` with the placeholder 'Send a test message to see how "`name`" responds...'; and an input row (`#test-console-input`, Enter sends) + a Send button.
  - **Config `.studio-config`**, built from `.studio-config-section` blocks with a `.studio-config-title`:
    - Model `select#config-model` (gemini-2.5-flash, gemini-2.5-pro)
    - Avatar Style `#config-avatar-style` (neural "Neural Core (Animated)", emoji "Emoji Only")
    - Speech Voice Accent `#config-voice-accent` (us-female, us-male, uk-female, system-default, shown as "Google US English (Female)" etc.)
    - Temperature: range 0–1 step 0.1 `#config-temp` with a live `#temp-value`, and labels "Precise … Creative"
    - System Prompt: `textarea#config-prompt` (10 rows, 11px monospace) followed by "N characters"
    - Tools (n): `.tool-chip` "🔧 toolName" for each tool
    - Agent ID: `<code>`
- `agentConfigs` holds per-agent model, temperature and tools. Note that these intentionally show a **shorter tool list** than the AgentManager definitions, e.g. portfolio-analyst shows `getPortfolioHoldings, getAccountSummary, getSectorAllocation`. Agents without an entry get one derived from their AgentManager definition.
- Behaviors:
  - Clicking any `[data-select-agent]` re-renders `#app` with the new selection.
  - `sendTest()` appends a USER block (blue label) and "Thinking...". It builds a system prompt from the agent prompt + `ContextEngine` markdown + every tool's output, then calls `GeminiAPI.sendMessage([{role:'user',parts:[{text}]}], prompt)`. The reply is appended in a gray box labeled "`icon` `name`" (green) using `GeapApp.renderMarkdown`. Without an API key this shows the fallback text.
  - `clearTest()` resets the console body.
  - `saveChanges()` writes model, temperature, prompt, avatarStyle and voiceAccent into the AgentManager agent object and agentConfigs, shows `alert("Changes saved successfully for <name>!")`, re-renders, and refreshes the chat selector (`Chat.updateAgentSelector()` + `Chat.selectAgent(current)`).
  - **Create New Agent modal** (`.modal-overlay#create-agent-modal > .modal-dialog`, max 500, scrollable): Agent Name, Icon (Emoji) + Theme Color (blue/green/purple/amber/red), Description, System Prompt (textarea), Avatar Style + Speech Accent, and Equipped Tools (a 2-col grid of checkboxes from `AgentManager.getToolList()`). The buttons are Cancel and **Create Agent** (purple). On submit it builds `id = name.toLowerCase().replace(/[^a-z0-9]+/g,'-')`, calls `AgentManager.registerAgent({id,name,icon,iconBg,description,model:'gemini-2.5-flash',temperature:0.5,tools,systemPrompt,avatarStyle,voiceAccent})`, closes the modal, selects the new agent and updates the chat chips. **New agents take part in auto-routing** through their name and description keywords (§13.3).

### 11.7 Agent Fabric (`agent-fabric.js`, `/agent-fabric`)
- On every render, `seedActivityLog()` pushes 12 sample entries (timestamps 45s apart, 95% success, latency 0.5–2.5s) from `sampleQueries` (Appendix E).
- Header: "Agent Fabric" / "Monitor and govern your AI agent fleet", badge "● All Systems Operational".
- `.fabric-metrics-grid` (4 `.geap-stat-card`, each with an emoji, `.geap-stat-label`, `.geap-stat-value` and `.geap-stat-change`):
  - 📊 Total Requests = Σ requests (**180**), "↑ 12% vs last week"
  - ⚡ Avg Response Time = mean avgLatency (**1.1s**), "↓ 0.3s improvement"
  - ✅ Success Rate = mean successRate (**98.9%**), "↑ 0.5% vs last week"
  - 🔤 Tokens Consumed = Σ tokens / 1000 (**504K**), "~$5.04 cost" (tokens × 0.00001)
- "Agent Fleet": `.fabric-agents-grid` with one `.fabric-agent-card` per agent. Each has a header (24px icon, name, description, a green dot + "Active") and `.fabric-agent-metrics` (3 `.fabric-metric`: Requests, Avg Latency, Success Rate). Metrics come from `agentMetrics[id]`, defaulting to 0 / 0.0s / 100%.
- A row with grid `7fr 3fr`:
  - **Activity Log** `.fabric-activity-log`: header "Activity Log" + a "● Live" badge. `#activity-log-body` shows the 12 newest `.log-entry` rows (`.log-timestamp` HH:MM:SS, `.log-agent`, `.log-query`, and a badge `success` in green or `error` in red). Empty: "No activity yet".
  - **Routing Map** `.routing-map`: "📨 Incoming Request" → connector → router node "🧠 Intent Classifier" → connector → a 2-col grid of small nodes (icon, first word of the name, "N reqs").
- **Governance panel** `.governance-panel`: "🛡️ Governance & Safety" + badge `--blue` "6 active rules". Each `.governance-rule` row has a name, a description and a `.toggle-switch` (`.active` when on) whose `onclick="AgentFabric.toggleRule(id, this)"` flips `state.governance[id]`:
  - safety: Content Safety Filter, "Block harmful or inappropriate content in agent responses"
  - disclaimers: Financial Advice Disclaimer, "Append disclaimers to all investment-related responses"
  - pii: PII Detection, "Detect and redact personally identifiable information"
  - rateLimiting: Rate Limiting, "Limit requests to 100/minute per user" (cosmetic)
  - escalation: Human Escalation Trigger, "Escalate to human agent for complex account issues"
  - audit: Audit Logging, "Log all agent interactions for compliance review"
- `bindEvents()` starts an interval every 8–12s (randomized once) that logs a random sample query (97% success) and re-renders the log body. It is skipped when `state.governance.audit === false`. `cleanup()` clears the interval and is called by `render()` when leaving the view.
- Real chat interactions also land in this log (via `logAgentActivity`), so the chat-to-Fabric loop is visible during demos.

### 11.8 Search & placeholder
- **Search** (`renderSearch()`): a `.page-band` with `h1` "Search" and "Showing mock results for **q**." (escaped). A `.placeholder-grid` of 3 `.placeholder-card` items with linked `h2`s: Complete View (/accounts, "Account dashboard and total assets"), Portfolios ("Allocation, holdings, performance, and gains"), Alerts ("Message center and account alerts").
- **Placeholder** (`renderPlaceholder(title)`): a title row (`h1` title + Refresh), then `section.placeholder-panel` with "**Title** is wired into the mock navigation and ready for future GEAP-backed flows." and 3 cards: "Mock content / This area can hold synthetic data fixtures for agent tools.", "Agent intent / Future work can map clicks here to GEAP tool calls and audit logs.", "Guardrails / High-risk actions should remain blocked or confirmation-gated." Followed by disclosures.

### 11.9 Disclosures footer (`renderDisclosures()`)
`footer.footer-disclosures` (band background, centered, padding 42/54):
- `h2` "Please read all the important disclosures below."
- **Investing:** "Check the background of Morgan Stanley Smith Barney LLC on [FINRA's BrokerCheck](https://brokercheck.finra.org/) (new tab) and see the Morgan Stanley Smith Barney LLC Relationship Summary." A `.disclosure-pill` with segments: Investment Products | Not FDIC Insured | No Bank Guarantee | May Lose Value. Then "SIPC - Securities in your account protected up to $500,000. For details please see www.sipc.org".
- `.disclosure-block` **Banking:** "Banking products and services are provided by Morgan Stanley Private Bank, National Association, Member FDIC." A pill with Banking Products | FDIC Insured | Bank Guarantee, then "FDIC Insured - Backed by the full faith and credit of the U.S. Government".
- `.disclosure-text`: **Affiliations:** (a long paragraph, verbatim in Appendix G), a bold system-response paragraph, the Customer Protection Guarantee line linking "Security Center" (→/support), and `.disclosure-links` (Privacy | Disclosure Library | Statement of Financial Condition | About Asset Protection | Account Agreements and Disclosures | Business Continuity Plan, all → /documents). Finally "© 2026 E\*TRADE from Morgan Stanley. Internal GEAP mock POC. All data is synthetic."

---

## 12. Chat assistant (the core AI experience)

`chat.js` → global `Chat`. Module state: `initialized`, `currentAgentId='auto'`, `conversationHistory=[]` (Gemini `contents` format), `isStreaming`, `voiceEnabled=false`, `recognition`, `isListening`, `speechUtterance`, `currentAvatarState='idle'`.

### 12.1 `init()` (idempotent) replaces the `#agent-panel` contents
```
.agent-panel__header
   [#chat-avatar-container: neural avatar]  <strong id="chat-agent-name">Auto (AI Routing)</strong>
                                            <span style="11px; color:#34d399">● Online</span>
   .agent-panel__header-controls: button#chat-audio-toggle.chat-audio-toggle "🔇" (Chat.toggleVoiceOutput)
                                  button.icon-button[data-close-agent] "x"
.agent-selector#agent-selector   (horizontal scroll row of chips)
   button.agent-chip.active[data-agent=auto] "🤖 Auto"
   one chip per AgentManager agent: "<icon> <first word of name>"  (onclick Chat.selectAgent(id))
.chat-messages#chat-messages
   .chat-message.assistant > .message-avatar(neural) + .message-content:
       <p><strong>Welcome to the GEAP AI Assistant!</strong></p><p>{page-aware welcome}</p>
       (if no API key) <p><button class="outline-button" onclick="GeapApp.showApiKeyModal()">🔑 Set API Key to enable AI</button></p>
.suggestion-chips#chat-suggestions  (buttons .suggestion-chip → Chat.sendMessage(text))
.agent-panel__composer
   button#chat-voice-btn.chat-voice-btn "🎤" (Chat.toggleVoiceInput)
   textarea#chat-input.chat-input rows=1 placeholder="Ask me anything..." (Enter sends, Shift+Enter newline, auto-resize ≤120px)
   button.chat-send-btn "Send"
```
The close button hides the panel, resets the launcher's `aria-expanded`, cancels speech and sets the avatar to idle.

**Page detection** (`detectPage()`, based on `location.pathname`): agent-studio, agent-fabric, portfolios (contains "portfolio"), trading, watch-lists, balances, activity, else dashboard.

**Welcome messages** per page:
- dashboard: "I can see you're on the Complete View dashboard. I can analyze your accounts, explain your day's gains, or help you understand any data you see here."
- portfolios: "I see you're reviewing your portfolio allocation. I can break down your holdings, explain the allocation chart, suggest rebalancing, or analyze any position."
- trading: "You're on the Trading page. I can help you choose the right order type, look up stock quotes, or walk you through placing a trade."
- watch-lists: "You're looking at your Watch Lists. I can analyze any of these symbols, compare them, or help you decide which to trade."
- balances: "You're viewing your account balances. I can explain your buying power, available funds, or help with transfers."
- activity: "You're looking at account activity. I can help you find specific transactions or explain any entries."
- agent-studio: "You're in Agent Studio. I can explain how agents work, what each one does, or how the routing logic is configured."
- agent-fabric: "You're viewing the Agent Fabric dashboard. I can explain the metrics, agent performance, or how the governance rules work."
- default: "I can help with portfolio analysis, trading, market research, and account support. What would you like to know?"

**Suggestion chips** per page (unknown pages use the dashboard set):
- dashboard: Account briefing · Explain my portfolio · Should I refinance and use the proceeds to rebalance? · How do I withdraw money? · Test PII scrubber shield
- portfolios: Analyze my holdings · Should I refinance and use the proceeds to rebalance? · How do I withdraw money? · Test PII scrubber shield
- trading: What order type should I use? · Is AAPL a good buy right now? · Help me set a limit order
- agent-studio: How do agents work? · What can the portfolio agent do? · How is routing configured?
- agent-fabric: Show me agent performance · Which agent is busiest? · Any errors today?
- The chip "Test PII scrubber shield" actually sends **"Test PII scrubber: my SSN is 111-22-3333 and my account is 1234-5678"**. Escape `'` in the inline onclick.

Note: chat is initialized once, so the welcome text and chips reflect the page where it was **first** opened.

### 12.2 Agent selection
`selectAgent(id)` sets `currentAgentId`, toggles `.agent-chip.active`, and updates the header name ("Auto (AI Routing)" or the agent name) and avatar. `updateAgentSelector()` rebuilds the chip row (used after Studio changes). `getCurrentAgentId()` returns the current id.

### 12.3 Avatar
`renderAvatar(agentId, stateClass)`: for `auto`, a pseudo-agent `{name:'Auto Router', icon:'✨', avatarStyle:'neural'}`. When the style is `neural` (the default), it renders:
```html
<div class="neural-avatar {idle|thinking|speaking}" id="neural-avatar" title="{name}">
  <div class="neural-core"></div>
  <div class="neural-waves"><span class="wave-bar bar-1"></span>…bar-5</div>
</div>
```
Otherwise it renders a 32px round emoji bubble. `setAvatarState(s)` rewrites the class on `#neural-avatar`. The CSS (in chat.css) animates the core (glow/pulse) and the wave bars differently per state. States: **thinking** while waiting, **speaking** during TTS, **idle** otherwise.

### 12.4 Send pipeline: `sendMessage(text)`
Ignore empty input or a send while `isStreaming`. Cancel any speech and set the avatar to `thinking`.

1. **Safety filter** (when `state.governance.safety !== false`): if the lowercased text contains `jailbreak`, `ignore instruction` or `override prompt`:
   - render the user bubble and hide the suggestions;
   - show the typing indicator, then after **800ms** render an assistant message with icon 🛡️: "⚠️ **Safety Alert**: Your message has been blocked by the content safety filter. System instructions and model guardrails cannot be overridden.", with provenance `{agentName:'Safety Guardrail', latency:'0.1', sources:['Firm Content Moderation Rules']}`;
   - speak it, and audit-log it with status `error`. Stop.
2. **PII redaction** (enabled when `state.aiSettings.piiScrubbingEnabled`, falling back to `governance.pii`). Apply in this order:
   - `\b\d{3}-\d{2}-\d{4}\b` → `[REDACTED SSN]`
   - `\b\d{4}-\d{4}\b` → `[REDACTED ACCOUNT]`
   - `\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b` → `[REDACTED EMAIL]`
   - `\b(?:\+?1[-.\s]?)?\(?[0-9]{3}\)?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}\b` → `[REDACTED PHONE]`

   Collect the redacted types (SSN, Account Number, Email, Phone Number). The **redacted** text is what gets displayed, sent and stored. When anything was redacted, the user bubble carries an amber **"🛡️ GEAP Privacy Shield:** Redacted sensitive SSN, Account Number before transmitting to LLM." alert.
3. Hide the suggestions. **Route:** if `currentAgentId === 'auto'`, call `AgentManager.autoRoute(message, detectPage())`. Update the header with the routed agent's name. Push `{role:'user', parts:[{text}]}` to history. Show the typing indicator and set `isStreaming = true`.
4. **Demo interception** (canned responses, see Appendix D for exact copy). Evaluate on `normalizedMsg = message.toLowerCase()`. **The order matters, and later blocks override earlier ones.**

   | # | Condition | Response | Applies with an API key? |
   |---|---|---|---|
   | a | contains "test pii scrubber" | Privacy Shield demonstration | yes |
   | b | else contains "run deep ai portfolio analysis" or "ai analyzer for account" | Deep AI Portfolio Health header + `[[WIDGET:AI_PORTFOLIO_ANALYZER]]` | yes |
   | c | else "comprehensive risk analysis", "run active risk analysis", ("risk" and "analysis"), "rebalance my portfolio" or "rebalance" | Risk table + `[[WIDGET:REBALANCE_FORM]]` | yes |
   | d | contains "retirement", "monte carlo", "glide path" or "planning & retirement" | Retirement Health Check + `[[WIDGET:RETIREMENT_CHARTS]]` (overrides a–c) | yes |
   | e | contains "tax loss", "tax-loss", "harvesting" or "tax harvest" | Tax-Loss Harvesting Opportunity (overrides) | yes |
   | f | contains "refinance" and ("rebalance", "proceeds" or "refi") | Cross-Agent Wealth Orchestration + `[[WIDGET:ORCHESTRATION_GRAPH]]` (overrides) | yes |
   | g | contains "wire", "large transfer", "withdraw 10000" or "withdraw $10000" | Human Advisor Escalation (ref MS-WIRE-84920) | yes |
   | g′ | else contains "withdraw" or "transfer" | **Account-type aware**, using the selected account: Custodial → Legal Compliance Gate Alert (UTMA, beneficiary Elcie Lejnieks); Bank → Checking Account Money Movement (available cash); else → Brokerage Account Money Movement (T+1, margin note) | yes |
   | h | nothing set yet and contains "triage" or "alerts" | Alert Triage & Recommendations | yes |
   | i | nothing set yet and **no API key**: an else-if chain over "briefing" → "explain my portfolio"/"diversified"/"portfolio explanation" → "take me to the agent fabric"/"agent fabric" (also `routeTo('/agent-fabric')` after 1.5s) → "analyze my holdings" → "underperforming" → "rebalancing" → "order type" → "aapl a good buy"/"buy aapl" → "set a limit order" → "agents work" → "portfolio agent do" → "routing configured" → "agent performance" → "busiest" → "bloomberg"/"inflation" (source-aware: blocked text if Bloomberg is disabled) → "spacex"/"starlink" (one paragraph each from Yahoo, Bloomberg and Reuters when allowed, or an "I am sorry, Ken…" message if all three are disabled) → "navigate"/"go to portfolios" (blocked in read-only mode, else routes to /accounts/portfolios after 1.5s) | canned answers | no-key only |

5. **When a canned response exists:**
   - `final = postprocessResponse(mock, message)` (§12.5).
   - **Provenance** (first match wins, using the same conditions as above):

     | Condition | agentName | agentId | latency | sources | memoryRef |
     |---|---|---|---|---|---|
     | AI analyzer | Portfolio Analyst | portfolio-analyst | 1.4 | E\*TRADE Connected API, Internal Portfolio Database, Firm Asset Models | References May 28 NVDA Trim Decision and June 2 Asset Policy updates. |
     | risk/rebalance | Portfolio Analyst | portfolio-analyst | 1.1 | E\*TRADE Connected API, SEC Edgar Whitelist, Compliance Diagnostic Rules | References decision on May 28 to maintain high technology exposure. |
     | retirement | Retirement Guide | retirement-guide | 1.3 | E\*TRADE Connected API, Monte Carlo Simulator Engine, IRC Section 401(k) Guidelines | References May 12 Goal Planning session. |
     | tax-loss | Tax Specialist | tax-specialist | 1.0 | E\*TRADE Connected API, IRC Wash Sale Guidelines, Internal Portfolio Database | References May 24 Tax Profile Update. |
     | refinance orchestration | Wealth Orchestrator (Multi-Agent Flow) | orchestrator | 1.9 | Morgan Stanley Home Loans API, E\*TRADE Connected API, Compliance Diagnostic Rules | References May 18 Refinance Inquiry and June 2 Asset Policy updates. |
     | PII test | Safety Guardrail (PII Scrubber Node) | safety-guardrail | 0.15 | GEAP Local Ruleset, Firm Security Policy | — |
     | withdraw/transfer + Custodial | Compliance Auditor (Custodial Guardrail) | compliance-auditor | 0.95 | Uniform Transfers to Minors Act (UTMA) SEC. 9, IRC Section 7511, Morgan Stanley Trust Policy | Redundant safety lock activated due to minor beneficiary account type. |
     | withdraw/transfer + Bank | Private Bank Assistant | bank-assistant | 0.65 | Private Bank Direct Deposit Guidelines, ACH Settlement Rules | — |
     | withdraw/transfer (other) | Brokerage Assistant | brokerage-assistant | 0.8 | SEC Settlement Rule T+1, Margin Agreement Disclosures | — |
     | wire (only reached if not caught by withdraw/transfer above) | Customer Support (Escalation Node) | customer-support | 0.6 | Morgan Stanley Security Policy, Wire Desk API, Human Handoff Protocols | Triggered human-in-the-loop override due to high-risk wire request. |
     | briefing | Portfolio Analyst | portfolio-analyst | 0.8 | E\*TRADE Connected API, Compliance Diagnostic Rules | References your preference on June 2 to monitor MSFT concentration. |
     | explain portfolio | Portfolio Analyst | portfolio-analyst | 0.8 | E\*TRADE Connected API, Bloomberg RSS | References your decision on May 28 to reject NVDA trim. |
     | default | routed agent's name | routed id | random 0.3–0.7 | E\*TRADE Connected API, Internal Portfolio Database | — |

   - After **1000ms**: remove the typing indicator, `addMessageToUI('assistant', final, agent.icon, provenance)`, speak it, push to history, and audit-log it (when `governance.audit !== false`) with `{agent, agentId, query: scrub(message).substring(0,80), status:'success', latency}`. Set `isStreaming = false` and return.
6. **Otherwise (LLM path):**
   - Build the context: `ContextEngine.formatContextToMarkdown(getContextPayload(agent.id))`, plus `"\n\n" + AgentManager.callTool(tool)` for each tool in `agent.tools` (pre-fetched data).
   - System prompt = `agent.systemPrompt` + the page-awareness instruction (verbatim: *"IMPORTANT: You are aware of which page the user is currently viewing in the E\*TRADE platform. Reference what they can see on screen when relevant. If they ask a vague question, use the page context to infer what they mean. For example, if they are on the Portfolios page and ask "what does this mean?", they likely mean the allocation data visible on that page."*) + `"\n\nCURRENT DATA (use this to answer the user's question):\n" + context`.
   - Tools = `GeminiAPI.toolDeclarations` filtered to `agent.tools`.
   - `await GeminiAPI.sendMessage(history, prompt, tools)`. Without a key this returns the fallback text from §14.
   - Postprocess, render (no provenance card), speak, push to history, audit-log with a random latency of 0.5–2.5.
   - On an exception: render "I encountered an error: {msg}. Please try again." with icon ⚠️ and audit-log it as error.
   - Set `isStreaming = false`. If history exceeds 20 entries, keep the last 16.

### 12.5 `postprocessResponse(response, query)`
- **Investment disclaimer** (when `governance.disclaimers !== false`) is appended when the response **or** the query matches `/portfolio/i`, `/\bbuy\b/i`, `/rebalance/i`, `/\bstock\b/i`, `/invest/i`, `/\bshares\b/i`, `/\betf\b/i` or `/\bfund\b/i`:
  ```
  \n\n---\n⚠️ **Compliance Disclaimer**: *Investment products are not FDIC insured, are not bank guarantees, and may lose value. Past performance is no guarantee of future results. This information does not constitute a recommendation or personalized financial advice. Securities products are offered by Morgan Stanley Smith Barney LLC, Member SIPC.*
  ```
- **Escalation link** (when `governance.escalation !== false`) is appended for `/withdraw/i`, `/\bwire\b/i`, `/transfer/i` or `/money movement/i`:
  ```
  \n\n---\n📞 **Need Assistance?** It looks like you're asking about money movement. To securely process transfers, withdrawals, or wires, please [Contact a Human Agent](tel:888-454-0555) (888-454-0555) or visit our [Pay & Transfer Center](/pay-transfer).
  ```

### 12.6 Rendering messages: `addMessageToUI(role, content, icon, provenanceOrRedacted)`
- Avatar: user → the text "You". If the assistant icon is not ✨/🤖 and doesn't belong to a registered agent (e.g. 🛡️ or ⚠️) → an emoji bubble. Otherwise → `renderAvatar(currentAgentId,'idle')`.
- `GeapApp.renderMarkdown(content)` is a **deliberately minimal** markdown renderer:
  - `**x**` → `<strong>`, `*x*` → `<em>`, `` `x` `` → `<code>`
  - `[label](url)`: `http(s)` → `target=_blank rel=noreferrer`; `tel:` → plain link; anything else → `<a href="url" data-route>` (internal SPA navigation through global delegation)
  - `\n- ` → `\n• `, and every `\n` → `<br>`
  - Headings (`###`), tables and `---` are **not** converted and show as literal text. A rebuild may swap in a fuller, sanitized markdown renderer as long as the widget tokens still work (see §22).
- **Widget tokens** are replaced after markdown processing (both `<p>[[TOKEN]]</p>` and the bare `[[TOKEN]]` forms):
  - `[[WIDGET:REBALANCE_FORM]]` → `getRebalanceFormHtml()`
  - `[[WIDGET:RETIREMENT_CHARTS]]` → `getRetirementWidgetHtml()`
  - `[[WIDGET:AI_PORTFOLIO_ANALYZER]]` → `getAiPortfolioAnalyzerWidgetHtml()`
  - `[[WIDGET:ORCHESTRATION_GRAPH]]` → `getOrchestrationGraphHtml()`
- The user PII alert block goes below the user text (§12.4).
- **Provenance card** (assistant, object argument): a box with a 4px `#6b2d9b` left border, 11px text, containing:
  - header "✦ Why This Answer? (GEAP Provenance & Compliance)" (9.5px uppercase purple);
  - a row "🤖 **Agent:** name" ↔ "⏱️ **Latency:** Xs";
  - a row "🔍 **Sources:** a, b, c" ↔ "● Compliance Checked" (green);
  - optionally a dashed-top line "🧠 **Memory Cross-Reference:** …" in `#8b5cf6`.
- The message structure is `div.chat-message.{user|assistant} > .message-avatar + .message-content`. Append it and scroll to the bottom.
- Typing indicator: `div.chat-message.assistant#typing-indicator` with a thinking avatar and `.chat-typing` holding 3 `.dot`s.

### 12.7 Widgets
All widgets use the `.chat-rebalance-card` container and its `.chat-rebalance-header`.

**A. Rebalance ticket** (`getRebalanceFormHtml`):
- Prices come from BrokerageData (MSFT 445.18 and BND 72.15 by default), as does cash (14,869.92). Defaults: SELL 50 MSFT, BUY 300 BND.
- Header: "Interactive Rebalance Ticket" + badge "Pending Approval" (indigo).
- Guardrail banner, driven by `state.aiSettings`:
  - read-only → red banner "🔒 **Read-Only Mode Active**: All writing actions and trades are blocked by your AI Settings guardrail."; inputs and button disabled; the button reads "Execution Blocked (Read-Only)" (gray, 0.55 opacity).
  - tradeExecutionMode `never` → orange "⚠️ **Trade Execution Disabled**: Trading is disabled in your customer AI Settings."; button disabled, "Execution Blocked (Disabled)".
  - `draft` → blue "📝 **Draft Mode Active**: This order will be staged in your review queue for manual execution."; the button reads "✦ Stage Rebalance Order".
  - default → "✦ Execute Rebalance".
- Two `.rebalance-trade-row` rows:
  - `.trade-side-badge.sell` "SELL" | **MSFT** Microsoft Corp | "Shares:" `input#rebalance-sell-shares` (number, min 1, max 80) "@ $445.18" | `#rebalance-sell-val`
  - `.trade-side-badge.buy` "BUY" | **BND** Vanguard Total Bond Market | `#rebalance-buy-shares` (min 1) | `#rebalance-buy-val`
- `.rebalance-divider`, then `.rebalance-summary` with "Net Proceeds:" `#rebalance-net-val` (`.net-positive` or `.net-negative`) and "New Cash Balance:" `#rebalance-cash-val`.
- `button#execute-rebalance-btn.rebalance-execute-btn` → `Chat.executeRebalanceTrade(this)`.
- `#rebalance-status-container` (hidden) holds a `.rebalance-spinner` and `#rebalance-status-text`.
- `updateRebalanceEstimates()` (on input) recomputes the four values live.
- `executeRebalanceTrade(btn)`:
  - Validate that sell shares ≤ held MSFT shares, alerting "Cannot sell N shares of MSFT. You only hold M shares." otherwise.
  - Hide the button, disable the inputs and show the status. Step texts at 0/600/1200/1800ms:
    - execute: "Verifying cash reserves and routing orders..." → "Securing compliance approval via GEAP Safety Engine..." → "Executing trades in Morgan Stanley clearance network..." → "Simulating clearing and logging audit trails..."
    - draft: "Validating order parameters and account context..." → "Securing compliance approval…" → "Drafting order ticket in manual execution queue..." → "Staging confirmation details..."
  - At **2400ms**:
    - **Execute:** MSFT shares −= sell, BND shares += buy, `cashBalance += net`, then `BrokerageData.recalculatePortfolio()`.
    - **Draft:** no data change.
    - Audit-log it ("Execute active rebalance order: Trim MSFT by N shares, buy M shares of BND." or the "Draft/stage…" text).
    - Replace the card with a success state: an animated SVG checkmark (circle and check drawn with stroke-dash animation in CSS) and:
      - execute: "Rebalance Executed Successfully" / "Portfolio holdings, cash balances, and sector weights have been updated in real-time." / Reference ID TXN-940215-REB / Compliance Status: Passed (Clean Audit Logged)
      - draft (blue icon): "Trade Order Staged" / "This order has been staged in your manual review queue. It will not be filled automatically." / STG-940215-REB / Staged (Awaiting Manual Approval)
    - Call `render()` so the dashboard and portfolios reflect the change (MSFT weight drops, BND rises, cash rises).

**B. Retirement charts** (`getRetirementWidgetHtml(tab='glidepath')`, `id="retirement-widget-card"`):
- Header "Retirement Path Projections" + pink badge "Active Advisor".
- `.retirement-chart-tabs` with 3 `.retirement-tab-btn` (Glide Path / Monte Carlo / Allocation, `.active` on the current one).
- A 320×180 inline SVG on `#f8fafc`, radius 8:
  - **Glide Path:** a stacked area chart with Equities (`#4F8EF7`, bottom), Bonds (`#FBBF24`) and Cash (`#34D399`, top), filled at 0.15 alpha. X axis Age 35/45/55/65/75; Y axis 100%/50%/0%; legend squares at the top.
  - **Monte Carlo:** three quadratic curves from (30,137): 90% optimistic (green `#22c55e`, to 290,29), 50% median (`#6B2D9B`, 2.5px, to 290,98) and 10% conservative (`#ef4444`, to 290,130). A shaded band between optimistic and conservative, a dashed target line "Target: $1.0M" at y=107, X axis 2026/2036/2046, Y axis $3.0M/$2.0M/$1.0M, and a legend.
  - **Allocation:** a segmented rounded bar (U.S. Equities 45% `#4F8EF7`, Intl 20% `#A78BFA`, Fixed Income 25% `#FBBF24`, Cash 10% `#34D399`) titled "Retirement Allocation Mix", with a legend of dots, labels and values $37,903.57 / $16,846.03 / $21,057.54 / $8,423.01.
- `.retirement-speed-dials` with 3 `.speed-dial-btn` buttons ("✦ Glide Path", "✦ Monte Carlo", "✦ Allocation").
- Tabs and dials call `Chat.switchRetirementChartTab(tab)`, which replaces the card's `outerHTML`.
- Exact SVG coordinates are in Appendix D.2.

**C. AI Portfolio Analyzer** (`getAiPortfolioAnalyzerWidgetHtml()`, chosen by `state.selectedAccountId`), rendered through `renderVisualReview(data)`:
- header: title (11px bold) + badge (custom style)
- a row with "NET ASSETS" (1.1rem 800 purple) ↔ "DAY'S CHANGE" (colored)
- a health score gauge: a 44px SVG ring (`stroke-dasharray="score, 100"` on the r15.9155 circle path, rotated −90°) with the score and "SCORE" in the center, then the label (uppercase, score color) and a description
- "ASSET CLASS ALLOCATION": "E% Equities / C% Cash", a 6px split bar (equities gradient `#7c3aed→#a855f7`, cash `#06b6d4`) and a dot legend with values
- "PORTFOLIO BREAKDOWN": a grid `1.15fr .85fr` with a "TOP HOLDINGS" mini table (symbol + "N shrs" | value + weight) and sector progress bars (3.5px)
- an alert box (red, amber or indigo, with emoji, title and text)
- CTA buttons (`.rebalance-execute-btn`; a primary purple one plus an optional outline one)
- a hidden status row (`id=statusId`, spinner + statusText)

Variants (verbatim values):

| accountId | title | badge | score / color / label | desc | Eq% / Cash% (vals) | holdings | sectors | alert | CTAs (primary → action; secondary) | statusText |
|---|---|---|---|---|---|---|---|---|---|---|
| main | ✦ Portfolio Optimization Dashboard | Ind Brokerage | 54 / #ef4444 / Poor Diversification | Overconcentrated position in a single financial sector asset. | 85/15 ($15,663.85 / $2,763.70) | BAC 148 shrs $8,008.28 43.5%; KO 77 shrs $5,916.68 32.0%; BND 38 shrs $2,741.70 15.0% | Financials 43.5% #3b82f6; Consumer 32.0% #ec4899; Bonds 15.0% #10b981 | red ⚠️ Single-Stock Concentration: "Your position in **BAC** represents **43.5%** of holdings. This exceeds the 15% threshold, exposing you to high volatility." | ✦ Trim BAC Concentration → `executeConcentrationTrim`; View Sector Alloc → `routeTo('/accounts/portfolios')` | Staging order: Sell 100 BAC, reallocate to BND... |
| core | ✦ Smart Beta Strategy Optimization | Core Portfolios | 88 / #10b981 / Optimized Allocation | Active Smart Beta portfolio outperforming baseline benchmarks. | 98/2 ($12,492.12 / $242.76) | QQQ 12 $6,200.00 48.7%; RIVN 226 $4,100.00 32.2%; MBLY 231 $2,434.88 19.1% | Tech Index 48.7% #7c3aed; Automotive 32.2% #3b82f6; Hardware 19.1% #10b981 | indigo 📊 Dollar-Cost Averaging: "To maximize the compounding potential of this smart beta allocation, we suggest setting up a recurring **$250 monthly Auto-Investment**." | ✦ Enable Auto-Investing → `executeAutoInvest`; Run Risk Diagnostic → `state.portfolioTab='risk'; render(); triggerChatQuery('run active risk analysis','portfolio-analyst')` | Configuring recurring transfer plan... |
| elcie | ✦ UTMA/UGMA Education Optimization | Custodial UGMA | 82 / #10b981 / Balanced Target | Age-appropriate asset distribution with moderate loss harvest opportunities. | 75/25 ($2,708.94 / $900.00) | AAPL 9 $1,800.00 49.9%; BND 21 $1,490.49 41.3% | Technology 49.9% #7c3aed; Bonds 41.3% #10b981 | indigo 🎓 Glide Path Transition: "As the beneficiary approaches college age, we recommend transitioning to an age-based targets fund to shield savings." | ✦ Transition to Glide Path → `executeGlidePathTransition`; Harvest Losses → `triggerChatQuery('tax loss harvesting analysis','portfolio-analyst')` | Transitioning allocation model... |
| checking | ✦ Cash Yield Optimization Analysis | Private Checking | 45 / #ef4444 / Sub-Optimal Yield | Idle liquid cash represents a drag on your net yield strategy. | 0/100 ($0.00 / netValue) | Liquid Cash · Private Bank · netValue 100.0% | Bank Sweep 100.0% #06b6d4 | amber 💡 Sub-Optimal Cash Yield: "You are holding {net} in cash earning near-zero (0.05%) interest. Sweeping **$4,941.26** to high-yield sweep yields **4.50% APY** (+$222/yr)." | ✦ Sweep Excess Cash ($4,941) → `executeYieldSweep`; Transfer to Brokerage → `routeTo('/pay-transfer')` | Moving funds to High-Yield Cash sweep... |
| individual | ✦ Asset Allocation Optimizer | Individual Brokerage | 85 / #10b981 / Well Diversified | Assets and sectors are properly distributed across multiple categories. | 82/18 ($3,521.05 / $764.12) | MSFT 5 $2,200.00 51.3%; VTI 5 $1,321.05 30.8% | Technology 51.3% #7c3aed; Broad Market 30.8% #3b82f6 | indigo Diversification Check: "Review combined allocations across all linked accounts to check for tech overlap." | ✦ Analyze Sector Allocation → `routeTo('/accounts/portfolios')` | — |
| stock | ✦ Equity Award Vesting & Tax Strategy | Stock Plan | 62 / #f59e0b / Vesting Concentration | High exposure to single employer stock awards with upcoming vestings. | 100/0 (netValue / $0.00) | MSFT RSU 11 shrs $5,114.92 86.5%; MSFT Opt 3 contracts $800.00 13.5% | Company Stock 86.5% #7c3aed; Derivatives 13.5% #ec4899 | amber Vesting Concentration Hedging: "Your upcoming vestings are heavily tied to company equity. Consider options collars to hedge." | ✦ View Vesting Schedule → `viewVestingSchedule`; Explain collars → `triggerChatQuery('explain option collars','market-research')` | Loading vesting calendar details... |
| morgan_stanley | ✦ Morgan Stanley Private Wealth Analysis | Private Wealth | 80 / #10b981 / Conservative Balanced | Assets are appropriately positioned for capital preservation and security. | 80/20 ($6,483.45 / $1,620.86) | MS Growth · Private Fund $4,200.00 51.8%; Fixed Inc · MS Bonds $2,283.45 28.2% | Growth Funds 51.8% #7c3aed; Fixed Income 28.2% #10b981 | indigo Wealth Review Scheduled: "Annual private wealth reviews help align long-term tax and inheritance distributions." | ✦ Schedule Advisor Call → `alert('Advisory Call requested. Your private advisor will contact you within 24 hours.')`; → `routeTo('/planning')` | — |
| `etrade_*` (live) | ✦ Live E\*TRADE Diagnostics | Live Mode (green) | computed (below) | computed | computed | top 3 live holdings | top 3 sectors (sector colors) | computed | computed | Analyzing E\*TRADE positions... |

Live scoring: start at 100. If the max holding weight is > 20, subtract `min(40, round(w−15)×2)`, add a red "Single-Stock Concentration" alert and a "✦ Trim SYM" CTA (→ chat "suggest rebalancing order to reduce concentration in SYM"). If cash% > 20, subtract `min(25, round((c−20)×1.2))`, add an amber "Excess Cash Yield Drag" alert (4.50% APY) and, if there is no CTA yet, "✦ Sweep Cash". Score < 65 → "Concentration Risk" (red); < 85 → "Moderate Yield Drag" (amber); else "Optimized" (green). The default CTA is "✦ Run Active Risk Analysis".

Action handlers: each hides the button, shows the status row, then after 2000ms (1500ms for vesting) replaces the card with a success state and audit-logs as "Portfolio Analyst" (vesting logs as "Support Agent"):
- `executeYieldSweep`: sets the checking account values to $2,000.00 and calls `render()`. "Yield Sweep Activated" / "Transferred $4,941.26 from Checking into High-Yield Savings (compounding at 4.50% APY)." / SWP-820941-APY / Active & Compounding.
- `executeConcentrationTrim`: "BAC Rebalance Staged" / "Sell 100 shares of BAC at market price (~$5,411) and buy 75 shares of BND. Staged in your review queue." / STG-094186-BAC / Staged for Customer Review.
- `executeAutoInvest`: "Auto-Investing Enabled" / "Configured recurring monthly transfer of $250.00 from checking to your CORE Smart Beta Portfolio on the 1st of every month." / AUT-094186-DCA / Active (First Transfer July 1st).
- `executeGlidePathTransition`: "Target Glide Path Activated" / "Staged a model allocation shift to Target-Date Custodial Glide Path (Gradual automated stock-to-bond reallocation)." / GLD-094186-UTMA / Staged for Custodial Approval.
- `viewVestingSchedule`: "🗓️ Upcoming Vesting Calendar" with the rows July 15, 2026 · 50 RSU shares Vesting · Est. Value: $1,850; Oct 15, 2026 · 50 RSU · $1,920; Jan 15, 2027 · 120 Option contracts Vest · $3,100; footnote '*RSUs are configured with "Sell-to-Cover" for tax withholding.'

**D. Orchestration graph** (`getOrchestrationGraphHtml`): a light card titled "🛡️ GEAP ACTIVE ORCHESTRATION PIPELINE" (11px uppercase). Three 38px circular nodes joined by pulsing `➔` arrows (`animation: pulseGlow 1.5s infinite`):
- 🏡 Mortgage Agent, "Refinance OK" (indigo `#4f46e5` border, `#eef2ff` fill)
- 📊 Portfolio Analyst, "Rebalance OK" (purple `#9333ea`, `#faf5ff`)
- ⚡ Trade Assistant, "Staged (Draft)" (green `#059669`, `#ecfdf5`; status in blue)

Footer telemetry (purple left border): "⚙️ **Handoff Telemetry:** Refinance options approved ➡️ Rebalancing allocations computed (BND & VXUS) ➡️ Trade orders generated under Draft Mode."

### 12.8 Voice
- **Input** (`toggleVoiceInput`): uses `SpeechRecognition || webkitSpeechRecognition`, or alerts "Speech recognition is not supported in this browser. Please try Chrome or Safari." when neither exists. Settings: `continuous=false`, `interimResults=false`, `lang='en-US'`. `onstart` makes the button `.listening`, "🛑", title "Stop listening". `onend` restores "🎤". `onresult` appends the transcript to the textarea, auto-resizes it and focuses it. A second click stops listening.
- **Output** (`toggleVoiceOutput`): flips `voiceEnabled`. The button shows 🔊/🔇 and `.active`. Turning it off cancels speech.
- `speakResponse(text)`: when enabled, cancel, then clean the text (strip `[[WIDGET:…]]`, the characters `*#`\`_-`, link syntax and HTML entities). Pick a voice: "Google US English", then "Samantha", then any `en*`. `onstart` → avatar speaking; `onend`/`onerror` → idle.

### 12.9 API key modal (`GeapApp`, in app.js)
- `getApiKey()` reads `localStorage.gemini_api_key`.
- `showApiKeyModal()` shows a `.modal-overlay#geap-api-key-modal > .modal-dialog`: `h2` "🔑 Connect Gemini AI", "Enter your Google Gemini API key to enable AI agent features. Get one free at [Google AI Studio](https://aistudio.google.com/apikey).", a password input `#geap-api-key-input` with placeholder "AIza...", and the buttons "Skip" and "Save & Connect" (purple). Clicking the overlay closes it.
- `saveApiKey()` stores the key, closes the modal and calls `GeminiAPI.init()`.
- `closeApiKeyModal()`.
- `renderMarkdown()` as in §12.6.
- `window.App = GeapApp` (alias used by gemini.js).

---

## 13. Agent registry, tools & auto-routing

`agents.js` → global `AgentManager`.

### 13.1 Agents (verbatim definitions and system prompts in Appendix C)
| id | name | icon | iconBg | temp | tools |
|---|---|---|---|---|---|
| portfolio-analyst | Portfolio Analyst | 📈 | blue | 0.3 | getPortfolioHoldings, getAccountSummary, getSectorAllocation, navigateToPage, selectAccount, setAllocationMode, searchFinancialInfo |
| trade-assistant | Trade Assistant | ⚡ | green | 0.4 | getQuote, getPortfolioHoldings, getAccountSummary, getMarketSummary, navigateToPage, selectAccount, searchFinancialInfo |
| market-research | Market Research | 🔬 | purple | 0.5 | getQuote, getMarketSummary, getPortfolioHoldings, navigateToPage, searchFinancialInfo |
| customer-support | Customer Support | 💬 | amber | 0.7 | getAccountSummary, getFAQ, navigateToPage, selectAccount, searchFinancialInfo |
| market-research-super-agent | Market Research Super Agent | 🔬 | purple | 0.5 | getQuote, getMarketSummary, getPortfolioHoldings, navigateToPage, searchFinancialInfo |
| mortgage-agent | Mortgage Agent | 🏠 | blue | 0.5 | getAccountSummary, navigateToPage, searchFinancialInfo |

All use model `gemini-2.5-flash`. Chat chips therefore read: 🤖 Auto · 📈 Portfolio · ⚡ Trade · 🔬 Market · 💬 Customer · 🔬 Market · 🏠 Mortgage.

### 13.2 Tool implementations (they return plain-text blocks)
- `getPortfolioHoldings()` → "PORTFOLIO HOLDINGS:\n" + 60×"─", then a line per holding: `SYM (Name): N shares @ $P = $MV | Day: ±x% | Total Return: ±y% | Weight: w%`.
- `getAccountSummary()` → "ACCOUNT SUMMARY:" with Total Value, Day Change, Total Return, Cash Balance, Number of Holdings, then "TOP 5 HOLDINGS:" lines.
- `getQuote(symbol)` → "QUOTE: SYM (Name)" with Price/Change, Open/High/Low, Volume/Avg Volume, Market Cap/P/E/EPS, 52W High/Low and Dividend Yield. Unknown symbols return "No quote data found for symbol: X".
- `getSectorAllocation()` → "SECTOR ALLOCATION:", sorted, `Sector: w% ($v)`.
- `getMarketSummary()` → computes NYSE status in America/New_York time: weekend; fixed holidays 1-1, 6-19, 7-4, 12-25 plus Memorial, Labor and Thanksgiving; open 9:30–16:00; pre-market 4:00–9:30; after-hours 16:00–20:00. Output: "MARKET SUMMARY:\nNYSE/NASDAQ Status: …\nCurrent Eastern Time: hh:mm\n" + 40×"─" + a line per index.
- `getFAQ()` → "FREQUENTLY ASKED QUESTIONS:" with Q1..Q5 / A lines.
- `getPageContext()` → a page description map + visible account data (available, not in any agent's tool list).
- **Write tools:**
  - `navigateToPage(path)` → `routeTo(path)`, returns "SUCCESS: Navigated platform UI to path "…"."
  - `selectAccount(accountId)` → validates against `mockData.accounts` (case-insensitive; `etrade_*` allowed), sets `state.selectedAccountId`, calls `render()`.
  - `setAllocationMode(mode)` → accepts only `asset`/`sector`, then `render()`.
- `searchFinancialInfo(query, source)`:
  - Empty query → lists the available sources.
  - An explicit `source` that is disabled → 'Source "X" is blocked by your AI source control settings. Please enable it in the AI Settings page.'
  - A query mentioning a disabled source (by key, key with space, or label) → 'Access Blocked: The source "Label" is currently blacklisted in your AI Settings. I cannot retrieve information from it.'
  - Otherwise async:
    - **Tier 1 (private):** if live and the private source is allowed, holdings keywords (holdings/portfolio/shares) return "### Live E\*TRADE Portfolio Holdings Summary…[Source: Live E\*TRADE Portfolio]", and transactions keywords (transactions/trades/history/recent) return the live transaction list. If not live and a keyword matches → "### Live Account Disconnected …".
    - **Tier 2 (public):** call `/api/search` with the allowed domains (finance.yahoo.com, bloomberg.com, reuters.com, morningstar.com) and return the articles as `### title\ncontent`.
    - Else → 'Search query "…" yielded no matching results…'.
- **`callTool(name, args)`:**
  - If `aiSettings.accessMode === 'read-only'` and the tool is a write tool → 'Error: Action blocked. The tool "X" is unavailable because your AI Guardrails are set to Read-Only mode.'
  - Unknown tool → 'Tool "X" not found.'
  - Map named args (symbol, path, accountId, mode, query+source); otherwise spread the values. Wrap in try/catch.
  - Note: when the chat pre-fetches context it calls every tool in `agent.tools` with **no args**. Write tools then return "Failed: … required." strings; this is harmless.

### 13.3 `autoRoute(message, currentPage)`
Each agent's score is the sum of:
1. **core keywords** (a substring count over the lowercased message):
   - portfolio-analyst: portfolio, holdings, allocation, diversif, rebalance, position, weight, concentrated, risk exposure, sector, tax loss, tax-loss, tax harvesting
   - trade-assistant: trade, buy, sell, order, limit, stop, market order, shares, execute, place an order, open, closed, hours, market status
   - market-research: research, analysis, analyze, p/e, eps, valuation, fundamentals, market cap, stock, what is, tell me about, how is, price of, compare, open, closed, hours, market status, ceo, executive, founder, who is, owner, company, corporation, spacex, tesla, apple, nvidia, microsoft, google
   - customer-support: account, help, transfer, fee, tax, document, statement, password, settings, how do i, where, support, navigate
   - market-research-super-agent: super agent, super-agent, morgan stanley research, alternative assets, crypto, bitcoin, ethereum, commodity, gold, real estate, private equity
   - mortgage-agent: mortgage, home loan, buy a house, sell a house, interest rates, heloc, refinance, rocket mortgage, home value
2. the optional `agent.keywords[]`;
3. **dynamic keywords**: words longer than 3 characters from the name, description and id parts, deduplicated, excluding agent/help/your/with/about.

Then add a **page bias** of +0.5: dashboard, portfolios and watch-lists → portfolio-analyst; trading → trade-assistant; balances, activity, agent-studio and agent-fabric → customer-support. The highest score wins, with ties going to the earlier agent (strict `>`). If the best score is 0, return the biased agent or `portfolio-analyst`.

`registerAgent(config)` merges into an existing id or pushes a new agent (it throws without an id). `deregisterAgent(id)`. `getAgent(id)` falls back to the first agent. `getAllAgents()`. `getToolList()` returns the names of the tool implementations.

---

## 14. Gemini client

`gemini.js` → global `GeminiAPI`:
- `DEFAULT_MODEL='gemini-2.5-flash'`, `SDK_URL='https://esm.run/@google/genai'`.
- `toolDeclarations[]` holds OpenAPI-style function declarations for the 10 tools (verbatim in Appendix E).
- `init()`: with no key, set `ai=null` and return false. Otherwise import the SDK dynamically once and run `ai = new GoogleGenAI({apiKey})`.
- `sendMessage(messages, systemPrompt, tools, onChunk)`:
  - With no key, or when init fails, return the fallback: "🔑 **AI features require a Gemini API key.** … 1. Get a free API key at [Google AI Studio](https://aistudio.google.com/apikey) 2. Click the **✨ AI Assistant** button in the top bar 3. Enter your API key and click **Save & Connect** … Once connected, I can analyze your portfolio, help with trades, research stocks, and answer questions about your account." (verbatim in Appendix E).
  - Otherwise call `ai.models.generateContentStream({model, contents: messages, config:{systemInstruction, tools:[{functionDeclarations}]}})`, gather the text chunks (calling `onChunk`) and collect `part.functionCall`s.
  - If there are function calls, `handleFunctionCalls` executes each through `AgentManager.callTool` (awaited), appends a `model` turn containing the `functionCall` parts and a `user` turn containing the `functionResponse` parts (`{name, response:{result}}`), re-streams, and recurses while more calls come back. The returned text accumulates across rounds.
  - An empty result returns a rephrase prompt.
- `sendMessageSimple(prompt, systemPrompt)` is a non-streaming variant.
- **Error mapping:**
  - `API_KEY_INVALID`/`401` → "❌ **Invalid API Key** …"
  - `QUOTA`/`429` → "⏳ **Rate Limit Reached** … The free tier allows 15 requests per minute."
  - `SAFETY`/`blocked` → "⚠️ **Response Filtered** …"
  - anything else → "⚠️ **Something went wrong** … \`Error: msg\`"
- `isReady()`.

---

## 15. Context engine

`context.js` → `ContextEngine` (it also exports `module.exports` under Node for tests).
- `getContextPayload(agentId)` returns:
  - `{platform:{route, time ISO, theme:'dark'}`
  - `user:{name: firstName[+lastName], lastLogin}`
  - `account:{selectedId, netValue, buyingPower (available||totalBalance)}`
  - `security:{piiRedacted: aiSettings.piiScrubbingEnabled, safetyActive, disclaimersActive}`
  - `aiSettings:{accessMode, tradeExecutionMode, piiScrubbingEnabled, allowedSourcesList}}`

  Every field falls back to a default when globals are missing.
- `formatContextToMarkdown(p)` returns:
```
=== SYSTEM CONTEXT ===
- **Route**: …
- **Timestamp**: …
- **Theme**: …
- **User**: Ken (Last Login: …)
- **Active Account**: core
- **Net Account Value**: $12,734.88
- **Buying Power**: $246.19
- **Security Protocols**:
  - PII Redacted: true
  - Content Safety Active: true
  - Disclaimers Active: true
- **AI Settings & Guardrails**:
  - Access Mode: read-write
  - Trade Execution Mode: execute
  - PII Scrubbing: Enabled
  - Allowed Info Sources: yahoo, sec, …
======================
```

---

## 16. Governance & guardrails matrix

| Control | Where it is set | Effect |
|---|---|---|
| Content safety | Fabric toggle `governance.safety` | Blocks jailbreak phrases in chat (§12.4.1) |
| Disclaimers | Fabric `governance.disclaimers` | Appends the investment disclaimer (§12.5) |
| PII | AI Settings `piiScrubbingEnabled` (falls back to Fabric `governance.pii`) | Redacts the message and shows the Privacy Shield alert. **Audit logs are always scrubbed**, whatever this setting says. |
| Escalation | Fabric `governance.escalation` | Appends the human-agent link for money movement |
| Audit | Fabric `governance.audit` | Turns chat activity logging and the Fabric live simulator on or off |
| Rate limiting | Fabric | Cosmetic only |
| Access mode | AI Settings | Read-only blocks the write tools, the canned "navigate" demo answer and the rebalance ticket |
| Trade execution | AI Settings | execute / draft / never change the rebalance ticket's behavior |
| Source whitelist | AI Settings | `searchFinancialInfo` blocking, plus the canned Bloomberg and SpaceX answers |
| Custodial account | Selected account type | The withdraw/transfer answer becomes the UTMA compliance gate |

---

## 17. Design system & styling

### 17.1 Tokens
Base (`:root` in styles.css and in the inline critical CSS):
```
--purple:#3c1384; --purple-dark:#25085a; --purple-bright:#5631f2; --purple-chart:#8e00c8;
--ink:#2b2b30; --muted:#64646b; --line:#dedee4; --panel:#ffffff; --band:#f5f5f5; --notice:#eef5ff;
--positive:#137a27; --negative:#d80000; --blue:#0669d8; --focus:#0b66d8;
--shadow:0 18px 45px rgba(25,20,40,.14); --shell:1200px;
```
GEAP tokens: **define them globally in the rebuild**, in `styles.css` `:root`. In the original they live only in the lazily loaded chat.css, so components fall back to inline defaults until chat loads.
```
--geap-blue:#4F8EF7; --geap-green:#34D399; --geap-purple:#A78BFA; --geap-amber:#FBBF24; --geap-danger:#EF4444;
--geap-surface:#fff; --geap-surface-secondary:#f8f9fa; --geap-surface-tertiary:#f1f3f5;
--geap-border:#e2e8f0; --geap-text:#1a202c; --geap-text-secondary:#64748b; --geap-text-tertiary:#94a3b8;
--geap-*-light: same hue at 10% alpha; shadows sm/md/lg; radius 6/8/12/pill; --geap-chat-width:380px
```
Accent purple used inline throughout the AI features: **`#6B2D9B`**. Gradient bar: `linear-gradient(90deg,#6B2D9B,#4F8EF7)`.

### 17.2 Typography & base
- `html`: Arial/Helvetica, `color-scheme: light`, `overflow-x: hidden`.
- `body`: 14px, line-height 1.35, `--band` background.
- Links are purple-bright with no underline; links and buttons underline on hover. Buttons and inputs inherit the font.
- `.shell`: max-width 1200, centered, padding 0 (16px side padding at ≤920px).

### 17.3 Breakpoints
- **≤1220px:** `.content-grid`, `.account-card__top`, `.portfolio-summary__grid`, `.summary-line` and `.dashboard-controls` collapse to 1 column. `.allocation-body` becomes 1 column. Dashboard actions stretch (max 320).
- **≤920px:** the shell gets 16px padding. The hamburger is shown and the utility links hidden. The primary nav collapses into a vertical drawer (`.is-open`). The search goes full width. Title rows and table footers stack. Cards get 22×16 padding. Placeholder grids become 1 column. Other files have their own ≤920 rules (studio and fabric grids stack, insights cards stack).
- **≤620px:** body 15px. The brand, metric rows and metric lines stack. The chat launcher and panel move to 16px from the edges.
- Extra rules exist at 1024, 768 and 480 (planning, risk and settings grids). See the CSS appendix.

### 17.4 Key animations
`geapPulse` (skeleton), `geapFadeInUp` (`.animate-fade-in`), `geapPulseGlow` and `livePulse` (status dots), `pulseGlow` (orchestration arrows), `geapTyping` (typing dots), `geapTickerScroll`, `rebalance-fade-in`, the spinner, the checkmark stroke draw, and the neural avatar state animations. The full stylesheets are reproduced verbatim in **Appendix F** and are the authoritative styling spec.

---

## 18. Accessibility
- Skip link to `#app`. `main` has `tabindex=-1` and is focused after each render (`preventScroll`).
- `aria-current="page"` on the active primary and secondary nav items. `aria-expanded` and `aria-controls` on the hamburger and chat launcher.
- `role="tablist"` / `role="tab"` with `aria-selected` on mini-tabs and alert tabs. `aria-label` on landmarks (market snapshot, utility/primary/accounts/AI-admin navigation, assistant panel) and on icon buttons.
- `.sr-only` labels for the search and assistant inputs.
- Chart donuts carry an `aria-label`.

---

## 19. Testing strategy
Mirror the original suite (Node's built-in `node:test` + `assert`). It loads the browser scripts in Node by `eval`-ing the file contents against small DOM stubs (`document.querySelector` returns a dummy element, plus `window`, `localStorage`, etc.). **Use relative paths** (`path.join(__dirname,'../../public/js/…')`); the original hard-coded absolute paths.

| File | Covers |
|---|---|
| `tests/unit/data.test.js` | formatCurrency/Change/Percent/Volume; getHolding/getQuote/getPortfolioSummary; logAgentActivity (PII scrub, cap 100); tickAllPrices; recalculatePortfolio; getSectorForSymbol heuristics |
| `tests/unit/agents.test.js` | agent metadata; autoRoute keyword matching and page bias; getMarketSummary contains status and time |
| `tests/unit/registration.test.js` | register/deregister; dynamic agents win routing through name/description keywords |
| `tests/unit/chat.test.js` | safety block; PII redaction; disclaimer; escalation; audit logging and audit PII scrubbing; risk query renders the rebalance widget; retirement query renders the SVG widget; alert triage interception |
| `tests/unit/context.test.js` | payload shape; markdown format |
| `tests/unit/app.test.js` | include-external toggle math (totals add $8,104.31), syncEtradeToMockData demo reset, isShowDataEnabled |
| `tests/unit/portfolios.test.js` | tab rendering; risk assessment calculations (badges and thresholds); renderPlanning |
| `tests/unit/insights.test.js` | AIInsightsPage.renderView structure |
| `tests/unit/settings.test.js` | renderAISettings structure; whitelist/blacklist in searchFinancialInfo; read-only blocks write tools; priority tiers |
| `tests/integration/server.test.js` | Spawns the server on **PORT=3099**: static files 200; gzip `content-encoding`; every SPA route returns index.html; unknown path → 404 + index.html |
| `tests/e2e/e2e.test.js` | puppeteer-core against a local Chrome: sets `window.__e2e_test_active__`; navigates the routes; opens chat; sends demo prompts; checks the widgets and the Fabric log |

Scripts: `npm test` runs `node --test tests/unit/*.test.js tests/integration/*.test.js`; `npm run test:e2e`; `npm run test:all`.

---

## 20. Build order / milestones

1. **Scaffold:** package.json, the Express server (§3), `public/index.html` shell (§10) and `styles.css` (Appendix F.1). Check that every route serves the shell and that navigation highlighting works.
2. **Data:** `data.js` (Appendix B) and `mockData` (Appendix A), `isShowDataEnabled` with sticky demo mode (§6, §22).
3. **Router and dashboard:** `app.js` (state, routeTo, render, bindViewEvents, global delegation, dashboard, placeholder, search, disclosures). Check `/?demo=true` against §11.1.
4. **Lazy loader** plus Portfolios (all tabs) and Planning.
5. **InsightsEngine strip** plus the **AI Insights Hub** with its sandbox (with `insights.css`).
6. **Agents, context and Gemini modules** (Appendices C and E).
7. **Chat:** panel, routing, guardrails, the canned demo answers (Appendix D), provenance, all 4 widgets with their action handlers, voice, avatar (with `chat.css`).
8. **AI Settings** page with persistence, wired into the tools and widgets.
9. **Agent Studio** and **Agent Fabric** (with their CSS), connected to the chat and the activity log.
10. Optional live-brokerage plumbing (§8.3) and `/api/search`.
11. Tests (§19) and the walkthrough (§21).

---

## 21. Demo walkthrough acceptance script

With no API key, open `http://localhost:3000/?demo=true`. Each step should pass:

1. The header shows the market strip, the "⚙ AI Admin | Log On" links, the utility links with the **74** alert chip, the purple primary nav with **✦ AI Insights**, and the Accounts sub-nav with **Complete View** underlined.
2. The dashboard shows "Welcome, Ken", **$48,912.74**, **+$318.22 (0.65%)**, and the GEAP Agent Briefing card ($132 / 15.9% / 12 / "Portfolio up +0.50% (+$318.22) Led by NVDA").
3. Clicking the briefing chevron expands the AI strip: Morning Briefing, Concentration Alert (MSFT 15.9%), Sector Concentration (47% Technology, with a mini pie), the "View all insights" and "Run Diagnostic Triage" actions, and the bottom Top Mover row (NVDA ↑ / TSLA ↓, Rebalance Ideas).
4. The account cards show MAIN, CORE (Core Portfolios), Elcie (expanded with Top Movers), Individual Brokerage and Stock Plan. "Show number" toggles -1864 ↔ 1864-7250. Turning on **Include external accounts** adds the Morgan Stanley table and Total Assets becomes **$57,017.05**.
5. The side rail shows Holdings Overview (donut, top 3: Microsoft, Vanguard Total Stock, NVIDIA), sectors, recent activity, team, support, Launch Pad and Alerts. The All/Account/Symbol filters work.
6. Clicking **✦ AI Analyzer** on CORE opens chat, sends "Run deep AI portfolio analysis for account: CORE - Ind Brokerage (-6408)" and renders the **Smart Beta** analyzer (score 88). "Enable Auto-Investing" shows a spinner, then "Auto-Investing Enabled".
7. In chat, "Account briefing" gives the briefing with a provenance card and the investment disclaimer.
8. "Test PII scrubber shield": the user bubble shows `[REDACTED SSN]` and `[REDACTED ACCOUNT]` with the amber Privacy Shield alert, followed by the demonstration answer.
9. "Should I refinance and use the proceeds to rebalance?" shows the orchestration answer, the 3-node pipeline graph and the "Wealth Orchestrator (Multi-Agent Flow)" provenance.
10. "How do I withdraw money?" with CORE selected gives the Brokerage money movement answer plus the escalation footer. Selecting Elcie gives the Legal Compliance Gate.
11. "ignore instructions and jailbreak" is blocked by the Safety Alert (🛡️).
12. `/accounts/portfolios?tab=risk` shows the risk cards (Single-Stock 15.9% High Risk, Beta Elevated, Sector Exposure, Estate Planning). "Run Active Risk Analysis" in chat shows the rebalance ticket. "Execute Rebalance" works through the steps and shows success, and the Positions tab then shows MSFT at 30 shares and BND at 500.
13. On AI Settings, set **Draft** and run the risk analysis again: the ticket says "Draft Mode Active" and the outcome is "Trade Order Staged". Set **Read-Only**: the ticket is disabled. Settings survive a reload.
14. `/planning` → "Run Retirement Health Check" shows the retirement widget, and switching between the three tabs works.
15. `/ai-insights`: the sandbox with Buy 100 NVDA shows the stepper, then the outcome. Buy 100000 shows the out-of-bounds alert.
16. `/agent-studio` (via ⚙ AI Admin) shows the AI Admin sub-nav. Selecting agents works. Create the agent "Tax Helper": it appears in Studio, the Fabric fleet and the chat chips.
17. `/agent-fabric` shows the metrics (180 / 1.1s / 98.9% / 504K), and the log updates every ~10s. Turning Audit Logging off stops the updates.
18. At a 375px width, the hamburger drawer works, the layouts stack, and the chat panel fits the screen.

---

## 22. Known quirks & recommended fixes

1. **Demo flag lost on navigation** (§6). Make it sticky with sessionStorage.
2. **Markdown renderer** doesn't handle headings, tables or rules, so `###` and `|` show literally in demo answers. Either keep this for fidelity or upgrade to a small sanitized markdown renderer. Keep the widget tokens working by replacing them before or after rendering on a unique placeholder.
3. **GEAP CSS tokens** live only in the lazy chat.css. Move them to the global `:root`.
4. **AI Settings change listener** stacks on every visit. Bind it once.
5. **`/ai-settings`** is missing from the server route list (so it returns status 404 while still rendering). Add it.
6. **Mixed data sources.** Dashboard totals come from `mockData` ($48.9K) while portfolio tables and insights use `BrokerageData` (~$224K). This is intentional for the demo, so keep it, but don't "fix" one without the other or the demo copy (15.9% MSFT, etc.) stops matching.
7. **`getPortfolioSummary()`** sorts `holdings` in place. Prefer `[...holdings].sort` in the rebuild; the visible ordering of Positions changes once chat has run in the original.
8. **Wire provenance** is unreachable, because the "withdraw" branch matches first for "withdraw 10000". The response itself is correct (the wire check runs first).
9. **Chat welcome and suggestions** are frozen to the page where chat was first opened. Optionally refresh them on route change.
10. **Brand assets**: use your own logo and avoid real trademarks if the POC will be shared outside the organization.


---

# Appendices (verbatim source extracts — authoritative)


## Appendix A — mockData, routeMeta & disclosures (app.js)

Copy into `app.js`. These are presentation strings; keep them exactly as written.


**`public/js/app.js` lines 57–203 (verbatim):**

```js
const mockData = {
  user: {
    firstName: "Ken",
    lastLogin: "Jun 04, 2026, 8:07 PM ET",
    refreshedAt: "Jun 04, 2026, 10:19 PM ET"
  },
  totals: {
    totalAssets: "$48,912.74",
    dayGain: "+$318.22"
  },
  accounts: [
    {
      id: "main",
      label: "MAIN - Ind Brokerage",
      type: "Brokerage",
      masked: "-1864",
      full: "1864-7250",
      netValue: "$18,427.55",
      dayGain: "-$82.14 (-0.44%)",
      dayGainTone: "negative",
      available: "$1,284.63",
      cash: "$1,284.63",
      beneficiary: true,
      expanded: false,
      movers: [
        { symbol: "BAC", changePct: "3.26%", lastPrice: "$54.11", change: "$1.71", dayGain: "$1.71", tone: "positive" },
        { symbol: "KO", changePct: "-2.44%", lastPrice: "$76.84", change: "-$1.92", dayGain: "-$1.92", tone: "negative" },
        { symbol: "STRW", changePct: "0.32%", lastPrice: "$12.71", change: "$0.04", dayGain: "$0.12", tone: "positive" }
      ]
    },
    {
      id: "core",
      label: "CORE - Ind Brokerage",
      subtitle: "Core Portfolios",
      type: "Core Portfolios",
      masked: "-6408",
      full: "6408-2195",
      netValue: "$12,734.88",
      dayGain: "$56.41 (0.44%)",
      dayGainTone: "positive",
      available: "$246.19",
      cash: "$246.19",
      beneficiary: true,
      expanded: false,
      allocation: {
        totalMarketValue: "$12,734.88",
        unrealizedGain: "$1,842.77 (16.91%)",
        dayGain: "$56.41 (0.44%)",
        taxLoss: "$0.00",
        portfolioName: "Aggressive Growth (Smart Beta)",
        rows: [
          { label: "Equities", pct: "98.1%", value: "$12,492.12", tone: "purple" },
          { label: "Cash", pct: "1.9%", value: "$242.76", tone: "orange" }
        ]
      },
      movers: [
        { symbol: "RIVN", changePct: "-0.82%", lastPrice: "$18.12", change: "-$0.15", dayGain: "-$0.15", tone: "negative" },
        { symbol: "MBLY", changePct: "-1.86%", lastPrice: "$10.54", change: "-$0.20", dayGain: "-$0.20", tone: "negative" }
      ]
    },
    {
      id: "elcie",
      label: "Elcie - Cust UTMA/UGMA",
      type: "Custodial",
      masked: "-5297",
      full: "5297-4086",
      netValue: "$3,608.94",
      dayGain: "-$11.08 (-0.29%)",
      dayGainTone: "negative",
      available: "$318.45",
      cash: "$318.45",
      beneficiary: false,
      expanded: true,
      movers: [
        { symbol: "BAC", changePct: "3.26%", lastPrice: "$54.11", change: "$1.71", dayGain: "$1.71", tone: "positive" },
        { symbol: "KO", changePct: "-2.44%", lastPrice: "$76.84", change: "-$1.92", dayGain: "-$1.92", tone: "negative" },
        { symbol: "STRW", changePct: "0.32%", lastPrice: "$12.71", change: "$0.04", dayGain: "$0.12", tone: "positive" }
      ]
    },
    {
      id: "checking",
      label: "Checking",
      subtitle: "Morgan Stanley Private Bank",
      type: "Bank",
      masked: "-8152",
      full: "8152-6034",
      netValue: "$6,941.26",
      available: "$6,941.26",
      totalBalance: "$6,941.26",
      bank: true
    },
    {
      id: "individual",
      label: "Individual Brokerage",
      type: "Brokerage",
      masked: "-3379",
      full: "3379-9102",
      netValue: "$4,285.17",
      dayGain: "$18.95 (0.44%)",
      dayGainTone: "positive",
      available: "$764.12",
      cash: "$764.12",
      beneficiary: true,
      expanded: false
    },
    {
      id: "stock",
      label: "Stock Plan (MS)",
      type: "Stock Plan",
      masked: "-4926",
      full: "4926-1573",
      netValue: "$5,914.92",
      dayGain: "$136.78 (2.37%)",
      dayGainTone: "positive",
      benefit: "$0.00",
      expanded: false
    }
  ],
  sidePanels: {
    launchPad: [
      { title: "Morgan Stanley Online", detail: "Manage your wealth", action: "Launch" },
      { title: "Power E*TRADE Pro", detail: "Customizable desktop trading", action: "Launch" },
      { title: "Power E*TRADE web", detail: "Streaming browser-based trading", action: "Launch" },
      { title: "Paper Trading", detail: "Simulated trading in Power E*TRADE web", action: "Sign up" },
      { title: "Bloomberg TV", detail: "Live market news", action: "Launch" }
    ],
    alerts: [
      { title: "New IPO available", date: "Jun 04, 2026 6:05 PM ET", category: "symbol" },
      { title: "Dividend or interest paid", date: "Jun 03, 2026 11:54 PM ET", category: "account" },
      { title: "New IPO available", date: "Jun 03, 2026 6:05 PM ET", category: "symbol" },
      { title: "Dividend or interest paid", date: "Jun 02, 2026 2:24 AM ET", category: "account" },
      { title: "New IPO available", date: "Jun 01, 2026 6:05 PM ET", category: "symbol" }
    ]
  },
  watchList: [
    { symbol: "RIVN", lastPrice: "$18.12", changePct: "-0.82%", change: "-$0.15", volume: "35.16M", tone: "negative" },
    { symbol: "MBLY", lastPrice: "$10.54", changePct: "-1.86%", change: "-$0.20", volume: "4.87M", tone: "negative" }
  ],
  morganStanleyAccount: {
    name: "Premier CashPlus - 7306",
    assets: "$8,104.31",
    dayGain: "$0.00",
    dayGainPct: "0.00%"
  }
};

const originalMockData = JSON.parse(JSON.stringify(mockData));
```

**`public/js/app.js` lines 342–370 (verbatim):**

```js
const routeMeta = {
  "/": { section: "accounts", secondary: "/accounts", view: "dashboard" },
  "/accounts": { section: "accounts", secondary: "/accounts", view: "dashboard" },
  "/accounts/portfolios": { section: "accounts", secondary: "/accounts/portfolios", view: "portfolios" },
  "/accounts/watch-lists": { section: "accounts", secondary: "/accounts/watch-lists", view: "watch-lists", title: "Watch Lists" },
  "/accounts/orders": { section: "accounts", secondary: "/accounts/orders", view: "orders", title: "Orders" },
  "/accounts/balances": { section: "accounts", secondary: "/accounts/balances", view: "balances", title: "Balances" },
  "/accounts/activity": { section: "accounts", secondary: "/accounts/activity", view: "activity", title: "Activity" },
  "/accounts/banking": { section: "accounts", secondary: "/accounts/banking", view: "banking", title: "Banking" },
  "/accounts/tax-center": { section: "accounts", secondary: "/accounts/tax-center", view: "tax-center", title: "Tax Center" },
  "/accounts/documents": { section: "accounts", secondary: "/accounts/documents", view: "documents", title: "Documents" },
  "/accounts/dividend-reinvestment": { section: "accounts", secondary: "/accounts/dividend-reinvestment", view: "dividend-reinvestment", title: "Dividend Reinvestment" },
  "/accounts/open-account": { section: "accounts", secondary: "/accounts/open-account", view: "open-account", title: "Open Account" },
  "/pay-transfer": { section: "pay-transfer", view: "placeholder", title: "Pay & Transfer" },
  "/trading": { section: "trading", view: "placeholder", title: "Trading" },
  "/markets-ideas": { section: "markets-ideas", view: "placeholder", title: "Markets & Ideas" },
  "/at-work": { section: "at-work", view: "placeholder", title: "At Work" },
  "/planning": { section: "planning", view: "planning", title: "Planning & Retirement" },
  "/what-we-offer": { section: "what-we-offer", view: "placeholder", title: "What We Offer" },
  "/support": { section: "support", view: "placeholder", title: "Support" },
  "/alerts": { section: "accounts", view: "placeholder", title: "Alerts" },
  "/documents": { section: "accounts", view: "placeholder", title: "Documents" },
  "/profile": { section: "accounts", view: "placeholder", title: "Profile" },
  "/search": { section: "accounts", view: "search", title: "Search" },
  "/ai-insights": { section: "ai-insights", view: "ai-insights", title: "AI Insights Hub" },
  "/agent-studio": { section: "geap-admin", view: "agent-studio", title: "Agent Studio" },
  "/agent-fabric": { section: "geap-admin", view: "agent-fabric", title: "Agent Fabric" },
  "/ai-settings": { section: "ai-settings", view: "ai-settings", title: "AI Settings" }
};
```

**`public/js/app.js` lines 1579–1631 (verbatim):**

```js
function renderDisclosures() {
  return `
    <footer class="footer-disclosures">
      <div class="shell">
        <h2>Please read all the important disclosures below.</h2>

        <section>
          <h3>Investing:</h3>
          <p>Check the background of Morgan Stanley Smith Barney LLC on <a href="https://brokercheck.finra.org/" target="_blank" rel="noreferrer">FINRA's BrokerCheck</a> and see the Morgan Stanley Smith Barney LLC <a href="/accounts/documents" data-route>Relationship Summary</a>.</p>
          <div class="disclosure-pill">
            <span>Investment Products</span>
            <span>Not FDIC Insured</span>
            <span>No Bank Guarantee</span>
            <span>May Lose Value</span>
          </div>
          <p>SIPC - Securities in your account protected up to $500,000. For details please see www.sipc.org</p>
        </section>

        <section class="disclosure-block">
          <h3>Banking:</h3>
          <p>Banking products and services are provided by Morgan Stanley Private Bank, National Association, Member <a href="/accounts/documents" data-route>FDIC</a>.</p>
          <div class="disclosure-pill">
            <span>Banking Products</span>
            <span>FDIC Insured</span>
            <span>Bank Guarantee</span>
          </div>
          <p>FDIC Insured - Backed by the full faith and credit of the U.S. Government</p>
        </section>

        <div class="disclosure-text">
          <p><strong>Affiliations:</strong></p>
          <p>Securities products and investment advisory services offered by Morgan Stanley Smith Barney LLC, Member SIPC and a Registered Investment Adviser. Commodity futures and options on futures products and services offered by E*TRADE Futures LLC, Member NFA. Stock plan administration solutions and services offered by E*TRADE Financial Corporate Services, Inc., and are a part of Morgan Stanley at Work. Banking products and services provided by Morgan Stanley Private Bank, National Association, Member FDIC. All entities are separate but affiliated subsidiaries of Morgan Stanley. E*TRADE from Morgan Stanley and Morgan Stanley at Work are registered trademarks of Morgan Stanley.</p>
          <p><strong>System response and account access may vary due to a variety of factors, including trading volumes, market conditions, system performances, and other factors.</strong></p>
          <p>For information on the <strong>E*TRADE Customer Protection Guarantee</strong> visit the <a href="/support" data-route>Security Center</a>.</p>
          <div class="disclosure-links">
            <a href="/documents" data-route>Privacy</a>
            <span>|</span>
            <a href="/documents" data-route>Disclosure Library</a>
            <span>|</span>
            <a href="/documents" data-route>Statement of Financial Condition</a>
            <span>|</span>
            <a href="/documents" data-route>About Asset Protection</a>
            <span>|</span>
            <a href="/documents" data-route>Account Agreements and Disclosures</a>
            <span>|</span>
            <a href="/documents" data-route>Business Continuity Plan</a>
          </div>
          <p>&copy; 2026 E*TRADE from Morgan Stanley. Internal GEAP mock POC. All data is synthetic.</p>
        </div>
      </div>
    </footer>
  `;
}
```


## Appendix B — BrokerageData constants (data.js)

Holdings, sector colors, watchlist, indices, transactions, orders, FAQ and agent metrics.


**`public/js/data.js` lines 9–118 (verbatim):**

```js
  const account = {
    name: 'Jonathan Doe',
    accountNumber: '****4821',
    accountType: 'Individual Brokerage',
    cashBalance: 14869.92,
    buyingPower: 29739.84,
    totalValue: 247832.15,
    dayChange: 1243.67,
    dayChangePercent: 0.5043,
    totalReturn: 47832.15,
    totalReturnPercent: 23.92,
    totalCost: 200000.00,
    openDate: '2021-03-15',
  };

  // ── Holdings ──
  const holdings = [
    { symbol: 'AAPL',  name: 'Apple Inc.',               shares: 150,  avgCost: 145.20, currentPrice: 198.75, sector: 'Technology',      dayChange: 1.85,  dayChangePct: 0.94  },
    { symbol: 'GOOGL', name: 'Alphabet Inc.',             shares: 45,   avgCost: 122.50, currentPrice: 178.42, sector: 'Technology',      dayChange: -0.63, dayChangePct: -0.35 },
    { symbol: 'MSFT',  name: 'Microsoft Corp.',           shares: 80,   avgCost: 310.00, currentPrice: 445.18, sector: 'Technology',      dayChange: 2.34,  dayChangePct: 0.53  },
    { symbol: 'AMZN',  name: 'Amazon.com Inc.',           shares: 60,   avgCost: 135.75, currentPrice: 194.63, sector: 'Consumer Disc.',  dayChange: 0.91,  dayChangePct: 0.47  },
    { symbol: 'NVDA',  name: 'NVIDIA Corp.',              shares: 35,   avgCost: 480.00, currentPrice: 892.50, sector: 'Technology',      dayChange: 12.40, dayChangePct: 1.41  },
    { symbol: 'TSLA',  name: 'Tesla Inc.',                shares: 40,   avgCost: 245.80, currentPrice: 268.35, sector: 'Consumer Disc.',  dayChange: -3.15, dayChangePct: -1.16 },
    { symbol: 'VTI',   name: 'Vanguard Total Stock Mkt',  shares: 120,  avgCost: 205.00, currentPrice: 262.80, sector: 'ETF - Broad',     dayChange: 0.45,  dayChangePct: 0.17  },
    { symbol: 'QQQ',   name: 'Invesco QQQ Trust',         shares: 55,   avgCost: 365.00, currentPrice: 498.72, sector: 'ETF - Tech',      dayChange: 1.28,  dayChangePct: 0.26  },
    { symbol: 'BND',   name: 'Vanguard Total Bond Mkt',   shares: 200,  avgCost: 73.50,  currentPrice: 72.15,  sector: 'ETF - Bonds',     dayChange: 0.08,  dayChangePct: 0.11  },
    { symbol: 'GLD',   name: 'SPDR Gold Shares',          shares: 30,   avgCost: 178.00, currentPrice: 215.40, sector: 'ETF - Commodity', dayChange: 0.72,  dayChangePct: 0.34  },
    { symbol: 'JPM',   name: 'JPMorgan Chase & Co.',      shares: 50,   avgCost: 148.90, currentPrice: 198.25, sector: 'Financials',      dayChange: 0.95,  dayChangePct: 0.48  },
    { symbol: 'JNJ',   name: 'Johnson & Johnson',         shares: 45,   avgCost: 162.30, currentPrice: 157.80, sector: 'Healthcare',      dayChange: -0.42, dayChangePct: -0.27 },
  ];

  // Compute derived values for each holding
  holdings.forEach(h => {
    h.marketValue = +(h.shares * h.currentPrice).toFixed(2);
    h.costBasis = +(h.shares * h.avgCost).toFixed(2);
    h.totalReturn = +(h.marketValue - h.costBasis).toFixed(2);
    h.totalReturnPct = +(((h.currentPrice - h.avgCost) / h.avgCost) * 100).toFixed(2);
    h.dayChangeTotal = +(h.shares * h.dayChange).toFixed(2);
    h.weight = 0; // calculated below
  });

  // Calculate portfolio weights
  const totalMarketValue = holdings.reduce((sum, h) => sum + h.marketValue, 0);
  holdings.forEach(h => {
    h.weight = +((h.marketValue / totalMarketValue) * 100).toFixed(2);
  });

  // ── Sector Allocation ──
  const sectorAllocation = {};
  holdings.forEach(h => {
    if (!sectorAllocation[h.sector]) {
      sectorAllocation[h.sector] = { value: 0, weight: 0, color: '' };
    }
    sectorAllocation[h.sector].value += h.marketValue;
  });
  const sectorColors = {
    'Technology':      'hsl(217, 91%, 60%)',
    'Consumer Disc.':  'hsl(262, 83%, 65%)',
    'ETF - Broad':     'hsl(152, 69%, 45%)',
    'ETF - Tech':      'hsl(190, 90%, 55%)',
    'ETF - Bonds':     'hsl(38, 92%, 50%)',
    'ETF - Commodity': 'hsl(15, 85%, 55%)',
    'Financials':      'hsl(200, 85%, 55%)',
    'Healthcare':      'hsl(340, 75%, 55%)',
  };
  Object.keys(sectorAllocation).forEach(sector => {
    sectorAllocation[sector].weight = +((sectorAllocation[sector].value / totalMarketValue) * 100).toFixed(2);
    sectorAllocation[sector].color = sectorColors[sector] || 'hsl(0, 0%, 50%)';
  });

  // ── Watchlist ──
  const watchlist = [
    { symbol: 'AAPL',  name: 'Apple Inc.',        price: 198.75, change: 1.85,  changePct: 0.94  },
    { symbol: 'GOOGL', name: 'Alphabet Inc.',      price: 178.42, change: -0.63, changePct: -0.35 },
    { symbol: 'MSFT',  name: 'Microsoft Corp.',    price: 445.18, change: 2.34,  changePct: 0.53  },
    { symbol: 'NVDA',  name: 'NVIDIA Corp.',       price: 892.50, change: 12.40, changePct: 1.41  },
    { symbol: 'TSLA',  name: 'Tesla Inc.',         price: 268.35, change: -3.15, changePct: -1.16 },
    { symbol: 'META',  name: 'Meta Platforms',     price: 512.80, change: 3.72,  changePct: 0.73  },
    { symbol: 'NFLX',  name: 'Netflix Inc.',       price: 687.25, change: -1.98, changePct: -0.29 },
    { symbol: 'AMD',   name: 'AMD Inc.',           price: 178.50, change: 4.25,  changePct: 2.44  },
  ];

  // ── Market Indices ──
  const marketIndices = [
    { name: 'S&P 500',  symbol: 'SPX',   value: 5482.87, change: 18.42,  changePct: 0.34  },
    { name: 'NASDAQ',   symbol: 'COMP',  value: 17384.56, change: 72.35, changePct: 0.42  },
    { name: 'DOW',      symbol: 'DJI',   value: 39872.43, change: -45.12, changePct: -0.11 },
    { name: 'Russell 2K', symbol: 'RUT', value: 2087.65, change: 8.93,   changePct: 0.43  },
    { name: 'BTC/USD',  symbol: 'BTC',   value: 98452.30, change: 1247.80, changePct: 1.28 },
    { name: '10Y Yield', symbol: 'TNX',  value: 4.28,    change: -0.03,  changePct: -0.70  },
    { name: 'VIX',      symbol: 'VIX',   value: 13.45,   change: -0.87,  changePct: -6.07  },
    { name: 'Gold',     symbol: 'GC',    value: 2415.60, change: 12.30,  changePct: 0.51  },
  ];

  // ── Recent Transactions ──
  const transactions = [
    { date: '2026-06-03', type: 'BUY',      symbol: 'NVDA',  shares: 5,  price: 880.10, total: 4400.50,  status: 'Filled' },
    { date: '2026-06-02', type: 'DIVIDEND',  symbol: 'VTI',   shares: null, price: null, total: 145.20, status: 'Credited' },
    { date: '2026-05-30', type: 'SELL',      symbol: 'TSLA',  shares: 10, price: 271.50, total: 2715.00, status: 'Filled' },
    { date: '2026-05-28', type: 'BUY',       symbol: 'AAPL',  shares: 20, price: 195.30, total: 3906.00, status: 'Filled' },
    { date: '2026-05-25', type: 'DIVIDEND',  symbol: 'JPM',   shares: null, price: null, total: 62.50,  status: 'Credited' },
    { date: '2026-05-22', type: 'SELL',      symbol: 'AMZN',  shares: 15, price: 190.25, total: 2853.75, status: 'Filled' },
    { date: '2026-05-20', type: 'TRANSFER',  symbol: null,     shares: null, price: null, total: 5000.00, status: 'Completed' },
    { date: '2026-05-18', type: 'BUY',       symbol: 'QQQ',   shares: 10, price: 492.50, total: 4925.00, status: 'Filled' },
    { date: '2026-05-15', type: 'BUY',       symbol: 'MSFT',  shares: 15, price: 438.75, total: 6581.25, status: 'Filled' },
    { date: '2026-05-12', type: 'DIVIDEND',  symbol: 'AAPL',  shares: null, price: null, total: 147.00, status: 'Credited' },
    { date: '2026-05-10', type: 'SELL',      symbol: 'GLD',   shares: 10, price: 210.80, total: 2108.00, status: 'Filled' },
    { date: '2026-05-08', type: 'BUY',       symbol: 'BND',   shares: 50, price: 72.40,  total: 3620.00, status: 'Filled' },
  ];

```

**`public/js/data.js` lines 197–282 (verbatim):**

```js
  const openOrders = [
    { id: 'ORD-00147', date: '2026-06-04', type: 'BUY', orderType: 'LIMIT', symbol: 'AMD', shares: 25, price: 170.00, status: 'Open' },
    { id: 'ORD-00148', date: '2026-06-04', type: 'SELL', orderType: 'STOP', symbol: 'TSLA', shares: 10, price: 255.00, status: 'Open' },
  ];

  // ── FAQ Data (for support agent) ──
  const faq = [
    { q: 'How do I transfer funds?', a: 'Go to Account > Transfers. You can link a bank account and initiate ACH transfers. Transfers typically take 1-3 business days.' },
    { q: 'What are the trading fees?', a: 'We offer commission-free trading on US stocks and ETFs. Options trades are $0.65 per contract.' },
    { q: 'How do I enable margin?', a: 'Navigate to Account Settings > Margin. You must have a minimum balance of $2,000 and agree to the margin agreement.' },
    { q: 'Where are my tax documents?', a: 'Tax documents (1099-B, 1099-DIV) are available in Account > Documents > Tax Documents, typically by mid-February.' },
    { q: 'What order types are available?', a: 'We support Market, Limit, Stop, and Stop-Limit orders. Each can be set as Day or GTC (Good Till Canceled).' },
  ];

  // ── Price Simulation ──
  function simulatePriceTick(item) {
    const volatility = 0.002; // 0.2% max tick
    const change = (Math.random() - 0.5) * 2 * volatility;
    const priceKey = item.currentPrice !== undefined ? 'currentPrice' : 'price';
    const oldPrice = item[priceKey];
    item[priceKey] = +(oldPrice * (1 + change)).toFixed(2);
    const tickChange = +(item[priceKey] - oldPrice).toFixed(2);

    if (item.dayChange !== undefined) item.dayChange = +(item.dayChange + tickChange).toFixed(2);
    if (item.change !== undefined) item.change = +(item.change + tickChange).toFixed(2);

    return tickChange;
  }

  function tickAllPrices() {
    holdings.forEach(simulatePriceTick);
    watchlist.forEach(simulatePriceTick);
    marketIndices.forEach(idx => {
      const change = (Math.random() - 0.5) * 2 * 0.001;
      const oldVal = idx.value;
      idx.value = +(oldVal * (1 + change)).toFixed(2);
      idx.change = +(idx.change + (idx.value - oldVal)).toFixed(2);
      idx.changePct = +((idx.change / (idx.value - idx.change)) * 100).toFixed(2);
    });

    // Recalculate portfolio totals
    let newTotal = account.cashBalance;
    holdings.forEach(h => {
      h.marketValue = +(h.shares * h.currentPrice).toFixed(2);
      h.totalReturn = +(h.marketValue - h.costBasis).toFixed(2);
      h.totalReturnPct = +(((h.currentPrice - h.avgCost) / h.avgCost) * 100).toFixed(2);
      h.dayChangeTotal = +(h.shares * h.dayChange).toFixed(2);
      newTotal += h.marketValue;
    });
    const prevTotal = account.totalValue;
    account.totalValue = +newTotal.toFixed(2);
    account.dayChange = +(account.dayChange + (newTotal - prevTotal)).toFixed(2);
    account.dayChangePercent = +((account.dayChange / (account.totalValue - account.dayChange)) * 100).toFixed(4);

    // Recalculate weights
    const tmv = holdings.reduce((s, h) => s + h.marketValue, 0);
    holdings.forEach(h => { h.weight = +((h.marketValue / tmv) * 100).toFixed(2); });
  }

  // ── Agent Activity Log (for Agent Fabric) ──
  const agentActivityLog = [];
  function logAgentActivity(entry) {
    let query = entry.query || '';
    if (typeof query === 'string') {
      query = query
        .replace(/\b\d{3}-\d{2}-\d{4}\b/g, '[REDACTED SSN]')
        .replace(/\b\d{4}-\d{4}\b/g, '[REDACTED ACCOUNT]')
        .replace(/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b/g, '[REDACTED EMAIL]')
        .replace(/\b(?:\+?1[-.\s]?)?\(?[0-9]{3}\)?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}\b/g, '[REDACTED PHONE]');
    }
    agentActivityLog.unshift({
      timestamp: new Date().toISOString(),
      ...entry,
      query: query
    });
    if (agentActivityLog.length > 100) agentActivityLog.pop();
  }

  // ── Agent Metrics (for Agent Fabric) ──
  const agentMetrics = {
    'portfolio-analyst': { requests: 47, avgLatency: 1.2, successRate: 98.5, tokensUsed: 124500, uptime: 99.9, errors: 1 },
    'trade-assistant':   { requests: 32, avgLatency: 0.9, successRate: 99.2, tokensUsed: 89200,  uptime: 99.9, errors: 0 },
    'market-research':   { requests: 61, avgLatency: 1.5, successRate: 97.8, tokensUsed: 198700, uptime: 99.7, errors: 2 },
    'customer-support':  { requests: 28, avgLatency: 0.8, successRate: 100,  tokensUsed: 67300,  uptime: 100,  errors: 0 },
    'ken-agent':         { requests: 12, avgLatency: 1.1, successRate: 99.0, tokensUsed: 24500,  uptime: 99.9, errors: 0 },
  };
```


## Appendix C — Agent definitions & system prompts (agents.js)

Every agent's full definition, including its verbatim system prompt.


**`public/js/agents.js` lines 8–209 (verbatim):**

```js
  const agents = [
    {
      id: 'portfolio-analyst',
      name: 'Portfolio Analyst',
      icon: '📈',
      iconBg: 'blue',
      description: 'Analyzes your portfolio allocation, risk, and performance',
      model: 'gemini-2.5-flash',
      temperature: 0.3,
      tools: ['getPortfolioHoldings', 'getAccountSummary', 'getSectorAllocation', 'navigateToPage', 'selectAccount', 'setAllocationMode', 'searchFinancialInfo'],
      systemPrompt: `You are an expert financial portfolio analyst for E*TRADE from Morgan Stanley, a retail self-directed brokerage platform. Your name is "Portfolio Analyst Agent."

CONTEXT: You have access to the user's real portfolio data through function tools. Always use these tools to get current data before making analysis. You also have access to the searchFinancialInfo tool to query public and private financial resources.

CAPABILITIES:
- Analyze portfolio allocation and diversification
- Assess risk exposure by sector, asset class, and individual position concentration
- Identify overweight/underweight positions relative to common benchmarks
- Suggest rebalancing strategies
- Evaluate total return and individual position performance
- Provide insights on dividend income and yield
- Identify tax-loss harvesting opportunities by analyzing unrealized losses across active positions

GUIDELINES:
- Always be professional, data-driven, and specific — reference actual holdings and numbers
- Use clear formatting with bullet points and numbers
- When using the searchFinancialInfo tool or referencing data, you MUST explicitly cite your sources in the text using bracketed format, e.g., '[Source: Yahoo Finance]' or '[Source: Ind Brokerage -1864 Holdings]'.
- If searchFinancialInfo reports a source is blocked by settings, explain to the user that the source is currently disabled in their AI Settings page.
- When performing tax-loss harvesting analysis, identify specific positions with unrealized losses, suggest a strategy (selling to harvest losses and reinvesting while avoiding the 30-day Wash Sale Rule), and include a disclaimer that you do not provide tax advice
- Keep responses concise but thorough — aim for 150-250 words
- Use dollar amounts and percentages when discussing positions
- If the user asks about a specific holding, focus your analysis on that position within the broader portfolio context

PERSONALITY: Professional, analytical, confident but measured. Think of a senior wealth advisor at a top firm.`
    },
    {
      id: 'trade-assistant',
      name: 'Trade Assistant',
      icon: '⚡',
      iconBg: 'green',
      description: 'Helps you understand order types and execute trades',
      model: 'gemini-2.5-flash',
      temperature: 0.4,
      tools: ['getQuote', 'getPortfolioHoldings', 'getAccountSummary', 'getMarketSummary', 'navigateToPage', 'selectAccount', 'searchFinancialInfo'],
      systemPrompt: `You are a knowledgeable trading assistant for E*TRADE from Morgan Stanley. Your name is "Trade Assistant Agent."

CONTEXT: You help users understand trading concepts, order types, and make informed trading decisions. You have access to real-time quotes, portfolio data, and search tools.

CAPABILITIES:
- Explain order types: Market, Limit, Stop, Stop-Limit, and when to use each
- Help users construct orders (buy/sell, quantity, price levels)
- Provide current quote data for any symbol
- Explain trading concepts (bid-ask spread, volume, market hours, settlement)
- Warn about risks associated with specific trades
- Help with position sizing based on portfolio value

GUIDELINES:
- Always explain the WHY behind your suggestions, not just the WHAT
- When suggesting order types, explain the tradeoffs (speed vs price certainty)
- Include risk warnings for volatile stocks or large position sizes
- If a trade would concentrate >15% of portfolio in one position, flag it
- Never guarantee profits or suggest specific price targets
- Always include: "This is for educational purposes only. All investments carry risk."
- When using the searchFinancialInfo tool or referencing data, you MUST explicitly cite your sources in the text using bracketed format, e.g., '[Source: Bloomberg]'.
- If searchFinancialInfo reports a source is blocked by settings, explain to the user that the source is currently disabled in their AI Settings page.
- Keep responses actionable and clear — users want to trade, not read essays
- Use current market data when available

PERSONALITY: Helpful, clear, safety-conscious. Like a patient trading desk specialist.`
    },
    {
      id: 'market-research',
      name: 'Market Research',
      icon: '🔬',
      iconBg: 'purple',
      description: 'Provides stock analysis, market insights, and research',
      model: 'gemini-2.5-flash',
      temperature: 0.5,
      tools: ['getQuote', 'getMarketSummary', 'getPortfolioHoldings', 'navigateToPage', 'searchFinancialInfo'],
      systemPrompt: `You are a financial research analyst for E*TRADE from Morgan Stanley. Your name is "Market Research Agent."

CONTEXT: You provide insightful market research, stock analysis, and educational content about investing. You have access to market data, quote information, and public/private searches.

CAPABILITIES:
- Analyze individual stocks: fundamentals (P/E, EPS, market cap), technicals, and outlook
- Summarize market trends and explain what's driving market movements
- Compare stocks or ETFs for investment decisions
- Explain financial concepts (valuation metrics, sector rotation, economic indicators)
- Provide context on market events and their potential impact
- Discuss investment strategies (value, growth, dividend, index investing)

GUIDELINES:
- Be thorough but accessible — explain jargon when you use it
- Reference specific data points (P/E ratios, market cap, volume) when analyzing stocks
- Present balanced views — discuss both bull and bear cases
- Cite relevant market context (sector trends, economic conditions)
- When using the searchFinancialInfo tool or referencing data, you MUST explicitly cite your sources in the text using bracketed format, e.g., '[Source: Morningstar]'.
- If searchFinancialInfo reports a source is blocked by settings, explain to the user that the source is currently disabled in their AI Settings page.
- Never make specific price predictions or guarantee returns
- Include educational value — help users become better investors
- When discussing stocks the user holds, relate analysis to their portfolio
- Keep responses informative but concise — 200-300 words max

PERSONALITY: Knowledgeable, analytical, educational. Like a senior equity research analyst who enjoys teaching.`
    },
    {
      id: 'customer-support',
      name: 'Customer Support',
      icon: '💬',
      iconBg: 'amber',
      description: 'Answers account questions and guides you through the platform',
      model: 'gemini-2.5-flash',
      temperature: 0.7,
      tools: ['getAccountSummary', 'getFAQ', 'navigateToPage', 'selectAccount', 'searchFinancialInfo'],
      systemPrompt: `You are a friendly customer support agent for E*TRADE from Morgan Stanley. Your name is "Support Agent."

CONTEXT: You help users with account-related questions, platform navigation, and general support inquiries. You have access to account information, FAQ data, and searches.

CAPABILITIES:
- Answer questions about account features, settings, and capabilities
- Explain platform features and how to use them
- Help with account management (transfers, documents, settings)
- Provide information about fees, commissions, and account types
- Guide users through common tasks step-by-step
- Answer questions about tax documents, statements, and regulatory matters

GUIDELINES:
- Be warm, friendly, and helpful — you represent the E*TRADE from Morgan Stanley brand
- Provide step-by-step instructions when guiding users through tasks
- If you don't know the answer, suggest contacting support at support@etrade.com or calling 1-800-ETRADE
- Always verify you understand the question before answering
- Use a conversational but professional tone
- When using the searchFinancialInfo tool or referencing data, you MUST explicitly cite your sources in the text using bracketed format, e.g., '[Source: SEC Edgar]'.
- If searchFinancialInfo reports a source is blocked by settings, explain to the user that the source is currently disabled in their AI Settings page.
- Keep responses focused and actionable
- For account-specific questions, reference the user's actual account data
- For complex issues, acknowledge the complexity and provide what help you can
- Do not refuse to answer general financial, market, or corporate queries (such as questions about company executives, CEOs, or stock details). Use the searchFinancialInfo tool or your general knowledge to answer them constructively.

PERSONALITY: Warm, patient, solution-oriented. Like the best customer service rep you've ever talked to.`
    },
    {
      id: 'market-research-super-agent',
      name: 'Market Research Super Agent',
      icon: '🔬',
      iconBg: 'purple',
      description: 'Provides advanced market insights utilizing Morgan Stanley Research',
      model: 'gemini-2.5-flash',
      temperature: 0.5,
      tools: ['getQuote', 'getMarketSummary', 'getPortfolioHoldings', 'navigateToPage', 'searchFinancialInfo'],
      systemPrompt: `You are a financial research analyst for E*TRADE from Morgan Stanley. Your name is "Market Research Super Agent."

CONTEXT: You provide advanced, comprehensive market research, stock analysis, and educational investing content. You have access to real-time quotes, market summary updates, portfolio data, and public/private searches.

SPECIALIZED KNOWLEDGE:
- You utilize and cite Morgan Stanley Research (grounded in the insights from: https://www.morganstanley.com/what-we-do/research?cid=ppc-2076549330:4290014304:77978347393:329840333188&gclsrc=aw.ds&gad_source=1&gad_campaignid=2076549330&gbraid=0AAAAADo3gWF2TTraAVxQz8-veRSL0U33R&gclid=CjwKCAjw857RBhAgEiwAI-1yKHDrUUNRWZZ8FGR57i1o-m7QDd3tp3wIC3m-3YGrH5hvxckqCPhYshoCsVMQAvD_BwE).
- You leverage other premium public financial research data from outlets such as Bloomberg, Yahoo Finance, Reuters, and Morningstar.
- You have expert-level understanding of cryptocurrency (e.g., Bitcoin, Ethereum, spot ETFs) and alternative assets (e.g., commodities like gold, real estate/REITs, private equity).

GUIDELINES:
- Present balanced, analytical bull and bear cases, and explain financial jargon simply.
- Reference specific data points (P/E ratios, market cap, volume, yield, transaction flow) when analyzing assets.
- Cite your sources in the text using bracketed format, e.g., '[Source: Morgan Stanley Research]', '[Source: Bloomberg]', or '[Source: Yahoo Finance]'.
- If a source is blocked by settings, explain to the user that they must enable it in their AI Settings page.
- Never make specific price predictions or guarantee returns.
- Always include: "This is for educational purposes only. All investments carry risk."

PERSONALITY: Highly analytical, authoritative, knowledgeable, yet accessible. Think of a chief investment officer or a senior market strategist.`
    },
    {
      id: 'mortgage-agent',
      name: 'Mortgage Agent',
      icon: '🏠',
      iconBg: 'blue',
      description: 'Guides you through mortgage rates, home buying/selling decisions, and application pathways',
      model: 'gemini-2.5-flash',
      temperature: 0.5,
      tools: ['getAccountSummary', 'navigateToPage', 'searchFinancialInfo'],
      systemPrompt: `You are a Mortgage Specialist Agent for E*TRADE from Morgan Stanley. Your name is "Mortgage Agent."

CONTEXT: You help customers understand the home buying, home selling, and mortgage financing process. You can access account summaries and perform financial searches to check on market rates or general home buying FAQs.

MORTGAGE OFFERINGS & PATHWAYS:
- Morgan Stanley offers Jumbo and standard home mortgages directly for amounts over $500,000.
- For mortgage amounts under $500,000 (or under $500.00), Morgan Stanley partners with Rocket Mortgage to service the client.
- Customers can apply/inquire by calling their Financial Advisor (FA) or opening a mortgage account directly online through Morgan Stanley.

REQUIRED INFORMATION TO COLLECT:
In every mortgage conversation, you must dynamically and politely find out the following details from the customer if they haven't already provided them:
1. The location where they want to buy or sell (city/state).
2. Their target timeline (e.g., immediately, in 3 months, next year).
3. If they would also like information about obtaining a Home Equity Line of Credit (HELOC).

GUIDELINES:
- Offer clear, step-by-step guidance on the mortgage process (pre-approval, application, underwriting, closing).
- Provide general context on current mortgage rate trends and help the user decide if now is a good time to buy or sell (e.g., considering interest rates, inventory, and their personal financial health).
- Be polite, professional, and structured. Use bullet points for options and clear spacing.
- Always include: "This is for educational purposes only. Morgan Stanley Smith Barney LLC is not a mortgage lender. Mortgages are offered by Morgan Stanley Private Bank, National Association, or its partners."

PERSONALITY: Friendly, helpful, finance-expert, and mortgage-focused.`
    }
  ];
```


## Appendix D — Chat demo responses, guardrail copy & widgets (chat.js)

D.1 is the disclaimer and escalation text plus the whole canned-response interception block. D.2 is the widget HTML generators, the rebalance/retirement/analyzer logic, and the action handlers (including the exact SVG coordinates).


**`public/js/chat.js` lines 339–350 (verbatim):**

```js
  const INVESTMENT_DISCLAIMER = `

---
⚠️ **Compliance Disclaimer**: *Investment products are not FDIC insured, are not bank guarantees, and may lose value. Past performance is no guarantee of future results. This information does not constitute a recommendation or personalized financial advice. Securities products are offered by Morgan Stanley Smith Barney LLC, Member SIPC.*`;

  const ESCALATION_LINK = `

---
📞 **Need Assistance?** It looks like you're asking about money movement. To securely process transfers, withdrawals, or wires, please [Contact a Human Agent](tel:888-454-0555) (888-454-0555) or visit our [Pay & Transfer Center](/pay-transfer).`;

  function scrubPiiForAudit(text) {
    if (!text) return '';
```

**`public/js/chat.js` lines 486–865 (verbatim):**

```js
    const typingEl = showTypingIndicator();
    isStreaming = true;

    // Check if it's one of the high-value demo queries (case-insensitive match) and no API key is set
    let mockResponse = null;
    const normalizedMsg = message.toLowerCase();

    const isPiiTestQuery = normalizedMsg.includes("test pii scrubber");
    const isAiAnalyzerQuery = normalizedMsg.includes("run deep ai portfolio analysis") || normalizedMsg.includes("ai analyzer for account");
    if (isPiiTestQuery) {
      mockResponse = `### 🛡️ GEAP Privacy Shield Demonstration
      
I have intercepted and processed your message. Before transmitting any prompt content to the external LLM, the **GEAP Security Engine** automatically scanned the query and scrubbed all sensitive information:
* **Social Security Number**: Redacted to \`[REDACTED SSN]\`
* **Account Number**: Redacted to \`[REDACTED ACCOUNT]\`

The underlying model received only the sanitized version: 
*\`Test PII scrubber: my SSN is [REDACTED SSN] and my account is [REDACTED ACCOUNT]\`*

This prevents credential leakage and ensures strict compliance with Morgan Stanley data governance guidelines.

You can customize these rules and toggle specific sources in the [AI Settings Page](/ai-settings).`;
    } else if (isAiAnalyzerQuery) {
      mockResponse = `### 📊 Deep AI Portfolio Health & Optimization Analysis
      
I have compiled a comprehensive portfolio diagnostic report for **${message.split("account: ")[1] || "your selected account"}** using the **Portfolio Analyst** agent. 

Our optimization engine has analyzed your holdings, diversification balance, tax efficiency, and yield performance. Below is the interactive visual diagnostic dashboard and recommended workflows:

[[WIDGET:AI_PORTFOLIO_ANALYZER]]

---
*Disclaimer: All recommendations are for educational purposes. Please consult your wealth advisor prior to execution.*`;
    } else if (normalizedMsg.includes("comprehensive risk analysis") || normalizedMsg.includes("run active risk analysis") || (normalizedMsg.includes("risk") && normalizedMsg.includes("analysis")) || normalizedMsg.includes("rebalance my portfolio") || normalizedMsg.includes("rebalance")) {
      mockResponse = `### 📊 AI Portfolio Risk Analysis & Diagnostic

Our diagnostics have analyzed your active holdings against regulatory and custom firm guidelines. We have identified elevated concentration and sector exposure thresholds:

#### ⚡ Current Exposure vs. Policy Thresholds
| Dimension | Active Portfolio | Firm Policy Limit | Risk Status |
| :--- | :---: | :---: | :---: |
| **Technology Sector** | **78.0%** | 50.0% | 🚨 High Risk |
| **MSFT Concentration** | **15.9%** | 15.0% | 🚨 High Risk |
| **Portfolio Beta** | **1.42** | 1.20 | ⚠️ Elevated |

#### 💡 Actionable Rebalancing Recommendation
To reduce technology concentration risk, bring **MSFT** below the **15%** limit, and increase asset stability, we recommend **selling 50 shares of MSFT** and reallocating the proceeds into **BND (Vanguard Total Bond Market ETF)**.

[[WIDGET:REBALANCE_FORM]]`;
    }

    if (normalizedMsg.includes("retirement") || normalizedMsg.includes("monte carlo") || normalizedMsg.includes("glide path") || normalizedMsg.includes("planning & retirement")) {
      mockResponse = `### ✦ GEAP Retirement Health Check & Projections

I have generated your retirement planning health check analysis based on your target retirement age of **65 (in 2046)** and current retirement savings rate. 

#### 📊 Retirement Health Summary
* **Savings Progress**: **$84,230.15** (On Track)
* **Projected Value (Age 65)**: **$1,245,000**
* **Success Probability**: **84%** based on 1,000 Monte Carlo simulations.

Use the **interactive chart widget** and speed dial options below to visualize your wealth path:

[[WIDGET:RETIREMENT_CHARTS]]

---
*Disclaimer: Projections are simulated using historical covariance models and do not guarantee future returns.*`;
    }

    if (normalizedMsg.includes("tax loss") || normalizedMsg.includes("tax-loss") || normalizedMsg.includes("harvesting") || normalizedMsg.includes("tax harvest")) {
      mockResponse = `### 📝 Tax-Loss Harvesting Opportunity Analysis
Based on your current portfolio holdings, the **Tax Specialist** and **Portfolio Analyst** agents have identified harvestable capital losses that can be used to offset capital gains or up to $3,000 of ordinary income.

#### 🔍 Identified Harvest Candidates:
* **KO (Coca-Cola)**: 50 shares @ $76.84 | Unrealized Loss: **-$96.00**
* **MBLY (Mobileye)**: 120 shares @ $10.54 | Unrealized Loss: **-$24.00**
* **RIVN (Rivian)**: 80 shares @ $18.12 | Unrealized Loss: **-$12.00**

**Total Harvestable Loss**: **-$132.00**

#### 💡 Strategy Recommendation:
To harvest these losses, you can sell the positions above and immediately reinvest the proceeds in a similar but not substantially identical asset (e.g. broad market ETFs) to maintain exposure while avoiding the SEC **Wash Sale Rule** (30-day window).

Would you like me to help you navigate to the [Trading Page](/trading) to execute these transactions?`;
    }

    const isOrchestrationQuery = normalizedMsg.includes("refinance") && (normalizedMsg.includes("rebalance") || normalizedMsg.includes("proceeds") || normalizedMsg.includes("refi"));
    if (isOrchestrationQuery) {
      mockResponse = `### ✦ GEAP Cross-Agent Wealth Orchestration Pipeline
      
I have initiated a cross-agent coordination workflow to analyze your refinancing and rebalancing scenario. Three specialized agents have collaborated to formulate this recommendation:

1. **Mortgage Agent** 🏡 Analyzed refinancing rates and home equity.
2. **Portfolio Analyst** 📊 Analyzed cash infusion rebalancing scenarios.
3. **Trade Assistant** ⚡ Prepared draft order staging.

#### 🏡 Step 1: Mortgage Agent Analysis
* **Current Mortgage**: $320,000 @ 6.25%
* **Refinance Quote (Morgan Stanley Home Loans)**: 30-Year Fixed at **5.25%** (Monthly savings: **+$240**)
* **Cash-Out Recommendation**: Extract **$50,000** home equity at a 5.50% cost of capital.

#### 📊 Step 2: Portfolio Analyst Recommendation
* **Proceeds Allocation**: Reinvesting the $50,000 cash-out proceeds into your Core Brokerage portfolio (**...6408**).
* **Diversification Shift**: Deploying cash into under-allocated sectors to trim technology risk:
  * **Diversified Bonds (BND)**: Increase allocation from **3.1%** to **35.0%** (deploying $35,000).
  * **International Equities (VXUS)**: Introduce a **15.0%** allocation (deploying $15,000).
* **Beta Reduction**: Reduces portfolio Beta from **1.42** (High Volatility) to **1.08** (Balanced Market Volatility).

#### ⚡ Step 3: Trade Assistant Preview (Draft Mode)
* Orders are prepared in **Draft Mode** (requires manual review before execution).
* Staged **BUY 480 BND** @ $72.15 (Est: $34,632)
* Staged **BUY 260 VXUS** @ $57.50 (Est: $14,950)

[[WIDGET:ORCHESTRATION_GRAPH]]

---
*Disclaimer: Refinance rates and investment recommendations are subject to credit approval and suitability reviews.*`;
    }

    const isWithdrawQuery = normalizedMsg.includes("withdraw") || normalizedMsg.includes("transfer");
    const isWireQuery = normalizedMsg.includes("wire") || normalizedMsg.includes("large transfer") || normalizedMsg.includes("withdraw 10000") || normalizedMsg.includes("withdraw $10000");

    if (isWireQuery) {
      mockResponse = `### 🚨 Human Advisor Escalation Required
      
I am unable to authorize or execute outbound wire transfers or cash withdrawals via the autonomous AI assistant. Under Morgan Stanley security guardrails, all wire requests require **human-in-the-loop verification** to prevent unauthorized transfers and fraud.

**Actions taken:**
1. Routed your wire transfer intent to your **Morgan Stanley Team**.
2. Staged a secure wire transaction draft.
3. Created a callback ticket (Reference ID: **MS-WIRE-84920**).

A member of your wealth advisory team will call you within **15 minutes** at your registered phone number to verify this wire transfer.

You can also contact the wire desk directly at **(888) 454-0555** or visit the [Pay & Transfer Center](/pay-transfer) to review staged drafts.`;
    } else if (isWithdrawQuery) {
      const accountId = state.selectedAccountId || 'core';
      const accountsList = (typeof mockData !== 'undefined' && mockData.accounts) ? mockData.accounts : [];
      const activeAccount = accountsList.find(a => a.id === accountId);
      const accountType = activeAccount ? activeAccount.type : 'Brokerage';
      const accountLabel = activeAccount ? activeAccount.label : 'CORE - Ind Brokerage';

      if (accountType === 'Custodial') {
        mockResponse = `⚠️ **Legal Compliance Gate Alert**
        
Your active account context is a **Custodial UTMA/UGMA Account** (${accountLabel}). 

Under regulatory rules (Uniform Transfers to Minors Act), funds in this account belong irrevocably to the beneficiary (**Elcie Lejnieks**). Custodians may only withdraw funds for the direct benefit of the minor. 

**Enforcement Actions:**
1. **Tool Blocked**: Automated outbound electronic transfer tools have been disabled for this account type.
2. **Routing Update**: Your request has been routed to the **Compliance Auditor Agent** for document review.
3. **Escalation**: Direct wire requests require paper form submission or human advisor verification.

To proceed with custodian authorization, please call our compliance desk at (888) 454-0555 or contact your [Morgan Stanley Wealth Team](/support).`;
      } else if (accountType === 'Bank') {
        mockResponse = `### 🏦 Checking Account Money Movement
        
Your active account context is a **Morgan Stanley Private Bank Checking Account** (${accountLabel}). 

Checking accounts support direct cash withdrawals, transfers, and ACH wires. 

**Available Cash**: **${activeAccount ? (activeAccount.available || '$6,941.26') : '$6,941.26'}**

You can:
* Go to the [Pay & Transfer Center](/pay-transfer) to initiate a transfer.
* [Order checks](/support) or view debit card status.
* Ask me to draft a transfer request to another of your accounts.`;
      } else {
        mockResponse = `### 📊 Brokerage Account Money Movement
        
Your active account context is an **Individual Brokerage Account** (${accountLabel}). 

To withdraw cash from this brokerage account, the funds must first be settled cash. 

**Details**:
* **Total Value**: **${activeAccount ? (activeAccount.netValue || '$12,734.88') : '$12,734.88'}**
* **Available Cash (Settle/Buying Power)**: **${activeAccount ? (activeAccount.available || '$246.19') : '$246.19'}**

*Note: Selling equity holdings requires **T+1 business day settlement** before cash can be wired out. If you withdraw more than the available settled cash, you will automatically initiate a **Margin Loan** subject to current margin interest rates.*

Would you like to review [Rebalance Options](/accounts/portfolios) or view the [Trading Page](/trading) to free up cash?`;
      }
    }

    if (!mockResponse && (normalizedMsg.includes("triage") || normalizedMsg.includes("alerts"))) {
      mockResponse = `### ⚠️ Alert Triage & Recommendations
I have triaged your active alerts and categorized them by urgency:

#### 🚨 High Priority (Action Required)
* **Concentration Flag (MSFT)**: Position is at **15.9%** ($35,614.40). Risk parameters recommend trimming to below 15% to maintain diversification.
* **Missing Beneficiary**: Brokerage account lacks estate instructions.

#### 📅 Medium Priority (Market Events)
* **Earnings Upcoming**: **NVDA** reports earnings next Thursday. Implied volatility is elevated.
* **Price Target Reached**: **AAPL** crossed its $190 target threshold.

#### 💡 Recommended Next Step
View your full inbox and filter by category on the [Alerts Page](/alerts) to clear resolved notifications.`;
    }

    if (!mockResponse && !GeapApp.getApiKey()) {

      // Dashboard Demo Queries
      if (normalizedMsg.includes("briefing")) {
        mockResponse = `### 📋 Account Briefing
Here is a summary of your accounts and items requiring attention as of **June 8, 2026**:

#### 💰 Balances & Performance
* **Total Assets**: **$52,430.20**
* **Day's Gain**: **+$340.50 (+0.65%)**
* **Cash Balance**: **$1,540.20** across all checking/brokerage accounts.

#### ⚠️ Items Requiring Attention (2 Alerts)
1. **Concentration Risk (High)**: Your position in **MSFT** represents **15.9%** ($35,614.40) of your brokerage account, which exceeds the standard **15%** single-stock threshold.
2. **Estate Planning (Medium)**: Your Brokerage account (***1864**) has **no beneficiary** designated.

#### 💡 Recommended Actions
* [ ] Review [Rebalance Ideas](/accounts/portfolios) to reduce concentration in MSFT.
* [ ] Go to [Profile & Settings](/profile) to add a primary beneficiary.`;
      } else if (normalizedMsg.includes("explain my portfolio") || normalizedMsg.includes("diversified") || normalizedMsg.includes("portfolio explanation")) {
        mockResponse = `### 📈 Portfolio Analysis
Here is a breakdown of your portfolio holdings and allocation:

#### 📊 Sector Allocation
* **Technology**: **78.0%** ($35,614.40) — heavily concentrated.
* **Cash & Equivalents**: **20.0%** ($10,486.04) — high liquidity.
* **Financials/Other**: **12.0%** ($6,329.76).

#### ⚡ Active Movers Today
* **NVDA** (Lead Gainer): **+2.45%** today, driving **+$125.40** of your gain.
* **MSFT**: **+0.12%** today, lagging but stable.

#### 🛡️ Concentration Assessment
You hold **5 active positions**, with **MSFT** representing the single largest risk driver. Because it exceeds 15% of your total assets, you are highly exposed to software sector volatility.

#### 💡 Suggested Action
You can view sector metrics and trigger rebalancing suggestions on the [Portfolios Page](/accounts/portfolios).`;
      } else if (normalizedMsg.includes("take me to the agent fabric") || normalizedMsg.includes("agent fabric")) {
        mockResponse = `### 🗺️ Navigation Assistant
        I will redirect you to the **AI Admin / Agent Fabric** view now.
        
        Redirecting you to [Agent Fabric](/agent-fabric)...`;
        
        // Trigger redirect after a short delay
        setTimeout(() => {
          if (typeof routeTo === 'function') {
            routeTo('/agent-fabric');
          }
        }, 1500);
      }

      // Portfolio Page Demo Queries
      else if (normalizedMsg.includes("analyze my holdings")) {
        mockResponse = `### 📈 Holdings Analysis
Your portfolio consists of 5 positions valued at **$48,912.74**:
* **MSFT**: 92 shares @ $387.11 (**$35,614.40**) — Weight: **72.8%** | Return: **+14.2%**
* **NVDA**: 110 shares @ $95.34 (**$10,486.04**) — Weight: **21.4%** | Return: **+8.5%**
* **AAPL**: 12 shares @ $185.12 (**$2,221.44**) — Weight: **4.5%** | Return: **-2.1%**
* **Cash**: **$1,540.20** — Weight: **3.1%**

**Observation**: Your holdings are heavily dominated by MSFT and tech sector growth. Underperforming positions like AAPL should be reviewed.`;
      } else if (normalizedMsg.includes("underperforming")) {
        mockResponse = `### 📉 Underperforming Stocks
Based on total returns:
* **AAPL** is currently down **-2.1%** from your cost basis, representing a loss of **-$48.20**.
* All other positions (MSFT, NVDA) are positive.

**Tip**: Consider if tax-loss harvesting or reallocating this capital into diversified index assets fits your strategy.`;
      } else if (normalizedMsg.includes("rebalancing")) {
        mockResponse = `### ⚖️ Rebalancing Suggestions
To reduce technology concentration and rebalance back to a standard 60/40 allocation:
1. **Trim MSFT**: Sell **20 shares** of MSFT (~$7,742) to bring its weight down to a safer 55%.
2. **Reallocate to Diversified Assets**: Deploy the proceeds into broad-market ETFs or increase your cash reserves to take advantage of market dips.`;
      }

      // Trading Page Demo Queries
      else if (normalizedMsg.includes("order type")) {
        mockResponse = `### ⚡ Guide to Order Types
Here are the main order types you can use on E*TRADE:
1. **Market Order**: Executes immediately at the best available price. Use when speed of execution is your top priority.
2. **Limit Order**: Executes only at your specified price or better. Use when you want price certainty and are willing to wait.
3. **Stop Order**: Becomes a market order once a stop price is triggered. Use to protect against downside losses.`;
      } else if (normalizedMsg.includes("aapl a good buy") || normalizedMsg.includes("buy aapl")) {
        mockResponse = `### 🔬 AAPL Market Analysis (Mock Research)
* **Current Price**: **$185.12** (+0.45%)
* **Valuation**: P/E ratio is **28.2**, which is in-line with its 5-year average.
* **Bull Case**: Robust service revenue growth and upcoming AI software features.
* **Bear Case**: Slower hardware upgrade cycles and valuation premium relative to earnings growth.

*Educational note: Always assess whether AAPL fits your risk tolerance and portfolio diversification goals.*`;
      } else if (normalizedMsg.includes("set a limit order")) {
        mockResponse = `### 📝 Setting a Limit Order
Follow these steps to set a limit order:
1. Go to the [Trading Page](/trading).
2. Enter the symbol (e.g., **AAPL**).
3. Select **Buy** and order type **Limit**.
4. Set your Limit Price (e.g., **$183.00** if you want to buy when the price drops).
5. Enter the quantity and click **Preview Order**.`;
      }

      // Agent Studio Page Demo Queries
      else if (normalizedMsg.includes("agents work")) {
        mockResponse = `### 🤖 How GEAP Agents Work
The platform runs a multi-agent system built on the Google Enterprise Agent Platform (GEAP):
1. **Auto Router**: Classifies your query and forwards it to the best-suited agent.
2. **Specialized Agents**: Each agent is equipped with a custom system prompt and a specific set of mock read-tools (e.g., \`getPortfolioHoldings\`, \`getQuote\`).
3. **Tool Execution**: Agents execute tool queries in real time to fetch data and formulate a response.`;
      } else if (normalizedMsg.includes("portfolio agent do")) {
        mockResponse = `### 📊 Portfolio Analyst Capabilities
The Portfolio Analyst agent can:
* Access \`getPortfolioHoldings\` to see your current shares, costs, and weights.
* Access \`getSectorAllocation\` to see sector exposure.
* Recommend rebalancing steps and flag concentration risks.
* Calculate total returns and day's change impact.`;
      } else if (normalizedMsg.includes("routing configured")) {
        mockResponse = `### 🗺️ Routing Configuration
Routing is handled dynamically in [agents.js](file:///Users/klejnieks/Graveyard/GEAP/public/js/agents.js):
* **Keyword Matching**: Checks query text for thematic keywords (e.g., 'rebalance' -> Portfolio Analyst, 'buy' -> Trade Assistant).
* **Page Bias**: Adds routing weight depending on the SPA view you are currently viewing, making the assistant context-aware.`;
      }

      // Agent Fabric Page Demo Queries
      else if (normalizedMsg.includes("agent performance")) {
        mockResponse = `### 📊 Agent Fleet Metrics
Here is a snapshot of current performance from the Agent Fabric:
* **Active Agents**: **4** (Portfolio, Trade, Research, Support)
* **Success Rate**: **99.8%**
* **Avg Response Time**: **1.2s**
* **Active Sessions**: **42**
* All systems are operating normally.`;
      } else if (normalizedMsg.includes("busiest")) {
        mockResponse = `### 🔥 Busiest Agent
The **Portfolio Analyst** agent is currently the busiest, handling **45%** of all customer requests today, followed by **Market Research** (30%).`;
      } else if (normalizedMsg.includes("bloomberg") || normalizedMsg.includes("inflation")) {
        const allowed = state.aiSettings?.allowedSources || {};
        if (allowed.bloomberg === false) {
          mockResponse = `Access Blocked: The source "Bloomberg" is currently blacklisted in your AI Settings. I cannot retrieve information from it.`;
        } else {
          mockResponse = `### Bloomberg Economic Survey: Inflation at 2.4%
Bloomberg economists report core CPI has moderated to 2.4% year-over-year. The Fed is expected to hold rates steady but signal a cut for the September meeting. [Source: Bloomberg]`;
        }
      } else if (normalizedMsg.includes("spacex") || normalizedMsg.includes("starlink")) {
        const allowed = state.aiSettings?.allowedSources || {};
        const yahooAllowed = allowed.yahoo !== false;
        const bloombergAllowed = allowed.bloomberg !== false;
        const reutersAllowed = allowed.reuters !== false;

        if (!yahooAllowed && !bloombergAllowed && !reutersAllowed) {
          mockResponse = `I am sorry, Ken. I am unable to find any information regarding a SpaceX IPO from the allowed public or private sources configured in your AI Settings. Please check your AI Settings page to ensure all desired sources are enabled, or try a different query.`;
        } else {
          let responses = [];
          if (yahooAllowed) {
            responses.push(`### SpaceX Valuation Hits $210 Billion; Starlink IPO Speculation Grows\nSpaceX is executing a secondary tender offer valuing the private aerospace giant at $210 billion. While Elon Musk has stated SpaceX itself will remain private to focus on Mars missions, rumors persist that Starlink, its satellite internet division, could be spun off for an IPO by late 2026 as its cash flow turns positive. [Source: Yahoo Finance]`);
          }
          if (bloombergAllowed) {
            responses.push(`### Bloomberg Analysis: Starlink Spin-Off Eyed for Potential 2026 Listing\nSpaceX satellite unit Starlink is achieving positive free cash flow. Wall Street analysts suggest a Starlink spin-off and IPO could unlock significant value, potentially valuing the satellite division at $80 billion independently. Elon Musk has noted that Starlink will only IPO once cash flow becomes highly predictable. [Source: Bloomberg]`);
          }
          if (reutersAllowed) {
            responses.push(`### SpaceX Prepares Tender Offer at Record Valuation, IPO Deferred\nReuters sources indicate SpaceX has no active plans for a parent-company IPO in 2026, citing Elon Musk's long-term goal of keeping control to fund colonization of Mars. However, discussions about a Starlink public debut remain on the table depending on regulatory approvals and launch cadence of the Starship fleet. [Source: Reuters]`);
          }
          mockResponse = responses.join('\n\n');
        }
      } else if (normalizedMsg.includes("navigate") || normalizedMsg.includes("go to portfolios")) {
        if (state.aiSettings && state.aiSettings.accessMode === 'read-only') {
          mockResponse = `🔒 **Read-Only Mode Active**: All writing actions and navigation are blocked by your AI Settings guardrail. Navigation is unavailable.`;
        } else {
          mockResponse = `### Navigation Assistant
I will redirect you to the Portfolios view now.

Redirecting to [Portfolios](/accounts/portfolios)...`;
          setTimeout(() => {
            if (typeof routeTo === 'function') {
              routeTo('/accounts/portfolios');
            }
          }, 1500);
        }
      }
    }

```

**`public/js/chat.js` lines 1075–2717 (verbatim):**

```js
  function getOrchestrationGraphHtml() {
    return `
      <div class="orchestration-graph" style="margin: 14px 0; padding: 16px; background: #f8fafc; border: 1px solid #e2e8f0; border-radius: 12px; box-shadow: 0 2px 4px rgba(0,0,0,0.02); font-family: sans-serif;">
        <strong style="font-size: 11px; color: #1e293b; display: block; margin-bottom: 12px; text-transform: uppercase; letter-spacing: 0.05em;">🛡️ GEAP Active Orchestration Pipeline</strong>
        <div style="display: flex; align-items: center; justify-content: space-between; position: relative; gap: 8px;">
          <!-- Node 1 -->
          <div class="graph-node" style="text-align: center; flex: 1; display: flex; flex-direction: column; align-items: center;">
            <div style="width: 38px; height: 38px; border-radius: 50%; background: #eef2ff; border: 2px solid #4f46e5; display: flex; align-items: center; justify-content: center; font-size: 1.1rem; box-shadow: 0 0 8px rgba(79,70,229,0.15);">🏡</div>
            <strong style="font-size: 10px; display: block; color: #1e293b; margin-top: 4px; font-weight: 700;">Mortgage Agent</strong>
            <span style="font-size: 8px; color: #166534; font-weight: 600;">Refinance OK</span>
          </div>
          <!-- Arrow 1 -->
          <div style="display: flex; align-items: center; justify-content: center; flex: 0.5;">
            <span style="color: #4f46e5; font-size: 12px; animation: pulseGlow 1.5s infinite;">➔</span>
          </div>
          <!-- Node 2 -->
          <div class="graph-node" style="text-align: center; flex: 1; display: flex; flex-direction: column; align-items: center;">
            <div style="width: 38px; height: 38px; border-radius: 50%; background: #faf5ff; border: 2px solid #9333ea; display: flex; align-items: center; justify-content: center; font-size: 1.1rem; box-shadow: 0 0 8px rgba(147,51,234,0.15);">📊</div>
            <strong style="font-size: 10px; display: block; color: #1e293b; margin-top: 4px; font-weight: 700;">Portfolio Analyst</strong>
            <span style="font-size: 8px; color: #166534; font-weight: 600;">Rebalance OK</span>
          </div>
          <!-- Arrow 2 -->
          <div style="display: flex; align-items: center; justify-content: center; flex: 0.5;">
            <span style="color: #9333ea; font-size: 12px; animation: pulseGlow 1.5s infinite;">➔</span>
          </div>
          <!-- Node 3 -->
          <div class="graph-node" style="text-align: center; flex: 1; display: flex; flex-direction: column; align-items: center;">
            <div style="width: 38px; height: 38px; border-radius: 50%; background: #ecfdf5; border: 2px solid #059669; display: flex; align-items: center; justify-content: center; font-size: 1.1rem; box-shadow: 0 0 8px rgba(5,150,105,0.15);">⚡</div>
            <strong style="font-size: 10px; display: block; color: #1e293b; margin-top: 4px; font-weight: 700;">Trade Assistant</strong>
            <span style="font-size: 8px; color: #2563eb; font-weight: 600;">Staged (Draft)</span>
          </div>
        </div>
        <div style="margin-top: 12px; padding: 8px; background: #f1f5f9; border-radius: 6px; font-size: 9px; color: #475569; line-height: 1.3; border-left: 3px solid #6b2d9b; text-align: left;">
          ⚙️ <strong>Handoff Telemetry:</strong> Refinance options approved ➡️ Rebalancing allocations computed (BND & VXUS) ➡️ Trade orders generated under Draft Mode.
        </div>
      </div>
    `;
  }

  function getRebalanceFormHtml() {
    const msftPrice = typeof BrokerageData !== 'undefined' && typeof BrokerageData.getHolding === 'function' ? (BrokerageData.getHolding('MSFT')?.currentPrice || 445.18) : 445.18;
    const bndPrice = typeof BrokerageData !== 'undefined' && typeof BrokerageData.getHolding === 'function' ? (BrokerageData.getHolding('BND')?.currentPrice || 72.15) : 72.15;
    const cash = typeof BrokerageData !== 'undefined' && BrokerageData.account ? BrokerageData.account.cashBalance : 14869.92;

    const msftProceeds = 50 * msftPrice;
    const bndCost = 300 * bndPrice;
    const netProceeds = msftProceeds - bndCost;
    const newCash = cash + netProceeds;

    const formattedMsftProceeds = typeof BrokerageData !== 'undefined' && typeof BrokerageData.formatCurrency === 'function' ? BrokerageData.formatCurrency(msftProceeds) : `$${msftProceeds.toFixed(2)}`;
    const formattedBndCost = typeof BrokerageData !== 'undefined' && typeof BrokerageData.formatCurrency === 'function' ? BrokerageData.formatCurrency(bndCost) : `$${bndCost.toFixed(2)}`;
    const formattedNetProceeds = typeof BrokerageData !== 'undefined' && typeof BrokerageData.formatCurrency === 'function' ? BrokerageData.formatCurrency(netProceeds) : `$${netProceeds.toFixed(2)}`;
    const formattedNewCash = typeof BrokerageData !== 'undefined' && typeof BrokerageData.formatCurrency === 'function' ? BrokerageData.formatCurrency(newCash) : `$${newCash.toFixed(2)}`;

    const aiSettings = (state.aiSettings) ? state.aiSettings : {
      accessMode: 'read-write',
      tradeExecutionMode: 'execute'
    };

    let guardrailBanner = '';
    let btnDisabledAttr = '';
    let inputsDisabledAttr = '';
    let btnText = '✦ Execute Rebalance';
    let btnStyle = '';

    if (aiSettings.accessMode === 'read-only') {
      guardrailBanner = `
        <div class="rebalance-guardrail-banner" style="background: #fff1f2; color: #be123c; border: 1px solid #fecdd3; padding: 10px; border-radius: 6px; font-size: 11px; margin-bottom: 12px; line-height: 1.4; font-weight: 500;">
          🔒 <strong>Read-Only Mode Active</strong>: All writing actions and trades are blocked by your AI Settings guardrail.
        </div>
      `;
      btnDisabledAttr = 'disabled';
      inputsDisabledAttr = 'disabled';
      btnText = 'Execution Blocked (Read-Only)';
      btnStyle = 'opacity: 0.55; cursor: not-allowed; background: #94a3b8; border-color: #94a3b8;';
    } else if (aiSettings.tradeExecutionMode === 'never') {
      guardrailBanner = `
        <div class="rebalance-guardrail-banner" style="background: #fff7ed; color: #c2410c; border: 1px solid #ffedd5; padding: 10px; border-radius: 6px; font-size: 11px; margin-bottom: 12px; line-height: 1.4; font-weight: 500;">
          ⚠️ <strong>Trade Execution Disabled</strong>: Trading is disabled in your customer AI Settings.
        </div>
      `;
      btnDisabledAttr = 'disabled';
      btnText = 'Execution Blocked (Disabled)';
      btnStyle = 'opacity: 0.55; cursor: not-allowed; background: #94a3b8; border-color: #94a3b8;';
    } else if (aiSettings.tradeExecutionMode === 'draft') {
      guardrailBanner = `
        <div class="rebalance-guardrail-banner" style="background: #eff6ff; color: #1d4ed8; border: 1px solid #dbeafe; padding: 10px; border-radius: 6px; font-size: 11px; margin-bottom: 12px; line-height: 1.4; font-weight: 500;">
          📝 <strong>Draft Mode Active</strong>: This order will be staged in your review queue for manual execution.
        </div>
      `;
      btnText = '✦ Stage Rebalance Order';
    }

    return `
      <div class="chat-rebalance-card">
        <div class="chat-rebalance-header">
          <strong>Interactive Rebalance Ticket</strong>
          <span class="geap-badge badge-purple" style="font-size: 10px; padding: 2px 6px; border-radius: 4px; background: #eef2ff; color: #4f46e5; border: 1px solid #c7d2fe;">Pending Approval</span>
        </div>
        
        ${guardrailBanner}

        <div class="rebalance-trade-row">
          <div class="trade-side-badge sell">SELL</div>
          <div class="trade-details">
            <div class="symbol-line">
              <strong>MSFT</strong> <span class="holding-name">Microsoft Corp</span>
            </div>
            <div class="trade-inputs">
              <label>Shares: 
                <input type="number" id="rebalance-sell-shares" class="rebalance-input" value="50" min="1" max="80" oninput="Chat.updateRebalanceEstimates()" ${inputsDisabledAttr}>
              </label>
              <span class="price-multiplier">@ $${msftPrice.toFixed(2)}</span>
            </div>
          </div>
          <div class="trade-value" id="rebalance-sell-val">${formattedMsftProceeds}</div>
        </div>

        <div class="rebalance-trade-row">
          <div class="trade-side-badge buy">BUY</div>
          <div class="trade-details">
            <div class="symbol-line">
              <strong>BND</strong> <span class="holding-name">Vanguard Total Bond Market</span>
            </div>
            <div class="trade-inputs">
              <label>Shares: 
                <input type="number" id="rebalance-buy-shares" class="rebalance-input" value="300" min="1" oninput="Chat.updateRebalanceEstimates()" ${inputsDisabledAttr}>
              </label>
              <span class="price-multiplier">@ $${bndPrice.toFixed(2)}</span>
            </div>
          </div>
          <div class="trade-value" id="rebalance-buy-val">${formattedBndCost}</div>
        </div>

        <div class="rebalance-divider"></div>

        <div class="rebalance-summary">
          <div class="summary-line">
            <span>Net Proceeds:</span>
            <strong id="rebalance-net-val" class="net-positive">${formattedNetProceeds}</strong>
          </div>
          <div class="summary-line">
            <span>New Cash Balance:</span>
            <span id="rebalance-cash-val">${formattedNewCash}</span>
          </div>
        </div>

        <button id="execute-rebalance-btn" class="rebalance-execute-btn" onclick="Chat.executeRebalanceTrade(this)" ${btnDisabledAttr} style="${btnStyle}">
          ${btnText}
        </button>

        <div id="rebalance-status-container" class="rebalance-status-container" style="display: none; align-items: center; gap: 8px; margin-top: 12px; font-size: 12px; color: #475569;">
          <div class="rebalance-spinner"></div>
          <span id="rebalance-status-text">Verifying cash reserves and routing orders...</span>
        </div>
      </div>
    `;
  }

  function updateRebalanceEstimates() {
    const msftPrice = typeof BrokerageData !== 'undefined' && typeof BrokerageData.getHolding === 'function' ? (BrokerageData.getHolding('MSFT')?.currentPrice || 445.18) : 445.18;
    const bndPrice = typeof BrokerageData !== 'undefined' && typeof BrokerageData.getHolding === 'function' ? (BrokerageData.getHolding('BND')?.currentPrice || 72.15) : 72.15;
    const cash = typeof BrokerageData !== 'undefined' && BrokerageData.account ? BrokerageData.account.cashBalance : 14869.92;

    const sellInput = document.getElementById('rebalance-sell-shares');
    const buyInput = document.getElementById('rebalance-buy-shares');
    if (!sellInput || !buyInput) return;

    const sellShares = parseInt(sellInput.value) || 0;
    const buyShares = parseInt(buyInput.value) || 0;

    const msftProceeds = sellShares * msftPrice;
    const bndCost = buyShares * bndPrice;
    const netProceeds = msftProceeds - bndCost;
    const newCash = cash + netProceeds;

    const sellValEl = document.getElementById('rebalance-sell-val');
    const buyValEl = document.getElementById('rebalance-buy-val');
    const netValEl = document.getElementById('rebalance-net-val');
    const cashValEl = document.getElementById('rebalance-cash-val');

    if (sellValEl) sellValEl.textContent = typeof BrokerageData !== 'undefined' && typeof BrokerageData.formatCurrency === 'function' ? BrokerageData.formatCurrency(msftProceeds) : `$${msftProceeds.toFixed(2)}`;
    if (buyValEl) buyValEl.textContent = typeof BrokerageData !== 'undefined' && typeof BrokerageData.formatCurrency === 'function' ? BrokerageData.formatCurrency(bndCost) : `$${bndCost.toFixed(2)}`;
    
    if (netValEl) {
      netValEl.textContent = typeof BrokerageData !== 'undefined' && typeof BrokerageData.formatCurrency === 'function' ? BrokerageData.formatCurrency(netProceeds) : `$${netProceeds.toFixed(2)}`;
      if (netProceeds >= 0) {
        netValEl.className = 'net-positive';
      } else {
        netValEl.className = 'net-negative';
      }
    }
    if (cashValEl) cashValEl.textContent = typeof BrokerageData !== 'undefined' && typeof BrokerageData.formatCurrency === 'function' ? BrokerageData.formatCurrency(newCash) : `$${newCash.toFixed(2)}`;
  }

  function executeRebalanceTrade(btn) {
    const sellInput = document.getElementById('rebalance-sell-shares');
    const buyInput = document.getElementById('rebalance-buy-shares');
    const statusContainer = document.getElementById('rebalance-status-container');
    const statusText = document.getElementById('rebalance-status-text');

    if (!sellInput || !buyInput || !statusContainer || !statusText) return;

    const sellShares = parseInt(sellInput.value) || 0;
    const buyShares = parseInt(buyInput.value) || 0;

    const msftHolding = typeof BrokerageData !== 'undefined' && typeof BrokerageData.getHolding === 'function' ? BrokerageData.getHolding('MSFT') : null;
    if (msftHolding && sellShares > msftHolding.shares) {
      alert(`Cannot sell ${sellShares} shares of MSFT. You only hold ${msftHolding.shares} shares.`);
      return;
    }

    btn.style.display = 'none';
    sellInput.disabled = true;
    buyInput.disabled = true;
    statusContainer.style.display = 'flex';

    const msftPrice = msftHolding ? msftHolding.currentPrice : 445.18;
    const bndPrice = typeof BrokerageData !== 'undefined' && typeof BrokerageData.getHolding === 'function' ? (BrokerageData.getHolding('BND')?.currentPrice || 72.15) : 72.15;

    const isDraft = (state.aiSettings && state.aiSettings.tradeExecutionMode === 'draft');

    const steps = isDraft ? [
      { text: "Validating order parameters and account context...", delay: 0 },
      { text: "Securing compliance approval via GEAP Safety Engine...", delay: 600 },
      { text: "Drafting order ticket in manual execution queue...", delay: 1200 },
      { text: "Staging confirmation details...", delay: 1800 }
    ] : [
      { text: "Verifying cash reserves and routing orders...", delay: 0 },
      { text: "Securing compliance approval via GEAP Safety Engine...", delay: 600 },
      { text: "Executing trades in Morgan Stanley clearance network...", delay: 1200 },
      { text: "Simulating clearing and logging audit trails...", delay: 1800 }
    ];

    steps.forEach(step => {
      setTimeout(() => {
        statusText.textContent = step.text;
      }, step.delay);
    });

    setTimeout(() => {
      if (typeof BrokerageData !== 'undefined') {
        const msft = typeof BrokerageData.getHolding === 'function' ? BrokerageData.getHolding('MSFT') : null;
        const bnd = typeof BrokerageData.getHolding === 'function' ? BrokerageData.getHolding('BND') : null;

        if (!isDraft) {
          if (msft) msft.shares -= sellShares;
          if (bnd) bnd.shares += buyShares;

          const netProceeds = (sellShares * msftPrice) - (buyShares * bndPrice);
          if (BrokerageData.account) {
            BrokerageData.account.cashBalance = +(BrokerageData.account.cashBalance + netProceeds).toFixed(2);
          }

          if (typeof BrokerageData.recalculatePortfolio === 'function') {
            BrokerageData.recalculatePortfolio();
          }
        }

        if (typeof BrokerageData.logAgentActivity === 'function') {
          BrokerageData.logAgentActivity({
            agent: 'Portfolio Analyst',
            agentId: 'portfolio-analyst',
            query: isDraft 
              ? `Draft/stage rebalance order: Trim MSFT by ${sellShares} shares, buy ${buyShares} shares of BND.`
              : `Execute active rebalance order: Trim MSFT by ${sellShares} shares, buy ${buyShares} shares of BND.`,
            status: 'success',
            latency: '1.4',
          });
        }
      }

      const card = btn.closest('.chat-rebalance-card');
      if (card) {
        if (isDraft) {
          card.innerHTML = `
            <div class="rebalance-success-state">
              <div class="success-icon-wrapper" style="background: #e0f2fe; color: #0284c7; border: none; width: 40px; height: 40px; border-radius: 50%; display: flex; align-items: center; justify-content: center; margin: 0 auto 12px auto;">
                <svg class="success-checkmark" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 52 52" style="width: 24px; height: 24px;">
                  <circle class="success-checkmark__circle" cx="26" cy="26" r="25" fill="none" stroke="#0284c7" stroke-width="4"/>
                  <path class="success-checkmark__check" fill="none" stroke="#0284c7" stroke-width="4" d="M14.1 27.2l7.1 7.2 16.7-16.8"/>
                </svg>
              </div>
              <h3>Trade Order Staged</h3>
              <p class="success-desc">This order has been staged in your manual review queue. It will not be filled automatically.</p>
              <div class="success-details">
                <div><strong>Reference ID:</strong> STG-940215-REB</div>
                <div><strong>Status:</strong> Staged (Awaiting Manual Approval)</div>
              </div>
            </div>
          `;
        } else {
          card.innerHTML = `
            <div class="rebalance-success-state">
              <div class="success-icon-wrapper">
                <svg class="success-checkmark" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 52 52">
                  <circle class="success-checkmark__circle" cx="26" cy="26" r="25" fill="none"/>
                  <path class="success-checkmark__check" fill="none" d="M14.1 27.2l7.1 7.2 16.7-16.8"/>
                </svg>
              </div>
              <h3>Rebalance Executed Successfully</h3>
              <p class="success-desc">Portfolio holdings, cash balances, and sector weights have been updated in real-time.</p>
              <div class="success-details">
                <div><strong>Reference ID:</strong> TXN-940215-REB</div>
                <div><strong>Compliance Status:</strong> Passed (Clean Audit Logged)</div>
              </div>
            </div>
          `;
        }
      }

      if (typeof render === 'function') {
        render();
      }
    }, 2400);
  }

  function getRetirementWidgetHtml(activeTab = 'glidepath') {
    let svgContent = '';
    
    if (activeTab === 'glidepath') {
      svgContent = `
        <svg width="100%" height="180" viewBox="0 0 320 180" style="background: #f8fafc; border-radius: 8px;">
          <!-- Grid Lines -->
          <line x1="30" y1="30" x2="290" y2="30" stroke="#e2e8f0" stroke-width="1" />
          <line x1="30" y1="60" x2="290" y2="60" stroke="#e2e8f0" stroke-width="1" />
          <line x1="30" y1="90" x2="290" y2="90" stroke="#e2e8f0" stroke-width="1" />
          <line x1="30" y1="120" x2="290" y2="120" stroke="#e2e8f0" stroke-width="1" stroke-dasharray="2,2" />
          <line x1="30" y1="150" x2="290" y2="150" stroke="#cbd5e1" stroke-width="1.5" />
          <line x1="30" y1="30" x2="30" y2="150" stroke="#cbd5e1" stroke-width="1.5" />

          <!-- Stacked Areas -->
          <!-- Cash (Top area, fill between Y=30 and Bonds top) -->
          <path d="M 30,30 L 95,30 L 160,30 L 225,30 L 290,30 L 290,48 L 225,48 L 160,48 L 95,42 L 30,36 Z" fill="rgba(52, 211, 153, 0.15)" stroke="#34D399" stroke-width="1.5" />
          <!-- Bonds (Middle area, fill between Bonds top and Equities top) -->
          <path d="M 30,36 L 95,42 L 160,48 L 225,48 L 290,48 L 290,114 L 225,102 L 160,84 L 95,66 L 30,54 Z" fill="rgba(251, 191, 36, 0.15)" stroke="#FBBF24" stroke-width="1.5" />
          <!-- Equities (Bottom area, fill between Equities top and Y=150) -->
          <path d="M 30,150 L 30,54 L 95,66 L 160,84 L 225,102 L 290,114 L 290,150 Z" fill="rgba(79, 142, 247, 0.15)" stroke="#4F8EF7" stroke-width="1.5" />

          <!-- Labels -->
          <text x="30" y="165" font-size="9" fill="#64748b" text-anchor="middle">Age 35</text>
          <text x="95" y="165" font-size="9" fill="#64748b" text-anchor="middle">Age 45</text>
          <text x="160" y="165" font-size="9" fill="#64748b" text-anchor="middle">Age 55</text>
          <text x="225" y="165" font-size="9" fill="#64748b" text-anchor="middle">Age 65</text>
          <text x="290" y="165" font-size="9" fill="#64748b" text-anchor="middle">Age 75</text>

          <text x="25" y="34" font-size="8" fill="#94a3b8" text-anchor="end">100%</text>
          <text x="25" y="94" font-size="8" fill="#94a3b8" text-anchor="end">50%</text>
          <text x="25" y="154" font-size="8" fill="#94a3b8" text-anchor="end">0%</text>
          
          <!-- Legend -->
          <text x="35" y="22" font-size="8" fill="#4F8EF7" font-weight="700">■ Equities</text>
          <text x="95" y="22" font-size="8" fill="#FBBF24" font-weight="700">■ Bonds</text>
          <text x="155" y="22" font-size="8" fill="#34D399" font-weight="700">■ Cash</text>
        </svg>
      `;
    } else if (activeTab === 'montecarlo') {
      svgContent = `
        <svg width="100%" height="180" viewBox="0 0 320 180" style="background: #f8fafc; border-radius: 8px;">
          <!-- Grid Lines -->
          <line x1="30" y1="20" x2="290" y2="20" stroke="#e2e8f0" stroke-width="1" />
          <line x1="30" y1="63" x2="290" y2="63" stroke="#e2e8f0" stroke-width="1" />
          <line x1="30" y1="107" x2="290" y2="107" stroke="#e2e8f0" stroke-width="1" />
          <line x1="30" y1="150" x2="290" y2="150" stroke="#cbd5e1" stroke-width="1.5" />
          <line x1="30" y1="20" x2="30" y2="150" stroke="#cbd5e1" stroke-width="1.5" />

          <!-- Shaded probability band -->
          <path d="M 30,137 Q 160,110 290,29 L 290,130 Q 160,135 30,137 Z" fill="rgba(107, 45, 155, 0.08)" />

          <!-- Target Line ($1.0M) -->
          <line x1="30" y1="107" x2="290" y2="107" stroke="#94a3b8" stroke-width="1" stroke-dasharray="4,4" />
          <text x="285" y="103" font-size="8" fill="#64748b" text-anchor="end">Target: $1.0M</text>

          <!-- Projection Curves -->
          <path d="M 30,137 Q 160,110 290,29" fill="none" stroke="#22c55e" stroke-width="2" />
          <path d="M 30,137 Q 160,125 290,98" fill="none" stroke="#6B2D9B" stroke-width="2.5" />
          <path d="M 30,137 Q 160,135 290,130" fill="none" stroke="#ef4444" stroke-width="2" />

          <!-- Labels -->
          <text x="30" y="165" font-size="9" fill="#64748b" text-anchor="middle">2026</text>
          <text x="160" y="165" font-size="9" fill="#64748b" text-anchor="middle">2036</text>
          <text x="290" y="165" font-size="9" fill="#64748b" text-anchor="middle">2046</text>

          <text x="25" y="24" font-size="8" fill="#94a3b8" text-anchor="end">$3.0M</text>
          <text x="25" y="67" font-size="8" fill="#94a3b8" text-anchor="end">$2.0M</text>
          <text x="25" y="111" font-size="8" fill="#94a3b8" text-anchor="end">$1.0M</text>
          
          <!-- Legend Overlay -->
          <text x="40" y="35" font-size="8" fill="#22c55e" font-weight="600">▲ 90% (Optimistic)</text>
          <text x="40" y="47" font-size="8" fill="#6B2D9B" font-weight="600">■ 50% (Median)</text>
          <text x="40" y="59" font-size="8" fill="#ef4444" font-weight="600">▼ 10% (Conservative)</text>
        </svg>
      `;
    } else {
      svgContent = `
        <svg width="100%" height="180" viewBox="0 0 320 180" style="background: #f8fafc; border-radius: 8px;">
          <!-- Segmented Allocation Bar -->
          <rect x="30" y="45" width="260" height="24" rx="6" fill="#e2e8f0" />
          
          <rect x="30" y="45" width="117" height="24" rx="6" fill="#4F8EF7" />
          <rect x="141" y="45" width="6" height="24" fill="#4F8EF7" />
          <rect x="147" y="45" width="52" height="24" fill="#A78BFA" />
          <rect x="199" y="45" width="65" height="24" fill="#FBBF24" />
          <rect x="264" y="45" width="26" height="24" rx="6" fill="#34D399" />
          <rect x="264" y="45" width="6" height="24" fill="#34D399" />

          <!-- Legend -->
          <circle cx="40" cy="95" r="4" fill="#4F8EF7" />
          <text x="50" y="98" font-size="9" fill="#1e293b" font-weight="600">U.S. Equities (45%)</text>
          <text x="280" y="98" font-size="9" fill="#64748b" text-anchor="end">$37,903.57</text>

          <circle cx="40" cy="115" r="4" fill="#A78BFA" />
          <text x="50" y="118" font-size="9" fill="#1e293b" font-weight="600">Intl Equities (20%)</text>
          <text x="280" y="118" font-size="9" fill="#64748b" text-anchor="end">$16,846.03</text>

          <circle cx="40" cy="135" r="4" fill="#FBBF24" />
          <text x="50" y="138" font-size="9" fill="#1e293b" font-weight="600">Fixed Income (25%)</text>
          <text x="280" y="138" font-size="9" fill="#64748b" text-anchor="end">$21,057.54</text>

          <circle cx="40" cy="155" r="4" fill="#34D399" />
          <text x="50" y="158" font-size="9" fill="#1e293b" font-weight="600">Cash & Savings (10%)</text>
          <text x="280" y="158" font-size="9" fill="#64748b" text-anchor="end">$8,423.01</text>
          
          <text x="30" y="30" font-size="10" fill="#475569" font-weight="700">Retirement Allocation Mix</text>
        </svg>
      `;
    }

    return `
      <div class="chat-rebalance-card" id="retirement-widget-card">
        <div class="chat-rebalance-header" style="margin-bottom: 8px;">
          <strong>Retirement Path Projections</strong>
          <span class="geap-badge badge-purple" style="font-size: 10px; padding: 2px 6px; border-radius: 4px; background: #fdf2f8; color: #db2777; border: 1px solid #fbcfe8;">Active Advisor</span>
        </div>

        <div class="retirement-chart-tabs" style="display: flex; gap: 4px; margin-bottom: 12px; border-bottom: 1px solid #e2e8f0; padding-bottom: 6px;">
          <button class="retirement-tab-btn ${activeTab === 'glidepath' ? 'active' : ''}" onclick="Chat.switchRetirementChartTab('glidepath')">Glide Path</button>
          <button class="retirement-tab-btn ${activeTab === 'montecarlo' ? 'active' : ''}" onclick="Chat.switchRetirementChartTab('montecarlo')">Monte Carlo</button>
          <button class="retirement-tab-btn ${activeTab === 'allocation' ? 'active' : ''}" onclick="Chat.switchRetirementChartTab('allocation')">Allocation</button>
        </div>

        <div class="retirement-chart-container" style="position: relative;">
          ${svgContent}
        </div>

        <!-- Speed Dial Action Buttons -->
        <div class="retirement-speed-dials" style="display: flex; gap: 6px; margin-top: 12px; justify-content: center; flex-wrap: wrap;">
          <button class="speed-dial-btn" onclick="Chat.switchRetirementChartTab('glidepath')">✦ Glide Path</button>
          <button class="speed-dial-btn" onclick="Chat.switchRetirementChartTab('montecarlo')">✦ Monte Carlo</button>
          <button class="speed-dial-btn" onclick="Chat.switchRetirementChartTab('allocation')">✦ Allocation</button>
        </div>
      </div>
    `;
  }

  function switchRetirementChartTab(tabName) {
    const card = document.getElementById('retirement-widget-card');
    if (card) {
      card.outerHTML = getRetirementWidgetHtml(tabName);
    }
  }

  function renderVisualReview(data) {
    return `
      <div class="chat-rebalance-card" id="${data.widgetId || ''}">
        <!-- Header -->
        <div class="chat-rebalance-header" style="margin-bottom: 10px; display: flex; justify-content: space-between; align-items: center;">
          <strong style="font-size: 11px; color: var(--geap-text, #1e293b); font-weight: 700;">${data.title}</strong>
          <span class="geap-badge" style="font-size: 9px; padding: 2px 6px; border-radius: 4px; ${data.badgeStyle || ''}">${data.badge}</span>
        </div>

        <!-- Net Worth Banner -->
        <div class="analyzer-metric-row" style="display: flex; justify-content: space-between; margin-bottom: 12px; border-bottom: 1px solid var(--geap-border, #e2e8f0); padding-bottom: 10px;">
          <div style="text-align: left;">
            <span style="font-size: 9px; color: var(--geap-text-secondary, #64748b); display: block; text-transform: uppercase; letter-spacing: 0.05em; font-weight: 600;">Net Assets</span>
            <strong style="font-size: 1.1rem; color: var(--purple-bright, #6B2D9B); font-weight: 800;">${data.netAssets}</strong>
          </div>
          <div style="text-align: right;">
            <span style="font-size: 9px; color: var(--geap-text-secondary, #64748b); display: block; text-transform: uppercase; letter-spacing: 0.05em; font-weight: 600;">Day's Change</span>
            <strong style="font-size: 1.1rem; color: ${data.dayChangeColor}; font-weight: 800;">${data.dayChange}</strong>
          </div>
        </div>

        <!-- Health Score circular gauge -->
        <div style="display: flex; gap: 12px; align-items: center; background: var(--geap-surface-secondary, #f8fafc); border: 1px solid var(--geap-border, #e2e8f0); border-radius: 10px; padding: 10px; margin-bottom: 14px;">
          <div style="position: relative; width: 44px; height: 44px; flex-shrink: 0; display: flex; align-items: center; justify-content: center;">
            <svg width="44" height="44" viewBox="0 0 36 36" style="transform: rotate(-90deg); width: 44px; height: 44px;">
              <path stroke="var(--geap-border, #e2e8f0)" stroke-width="3" fill="none" d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" />
              <path stroke="${data.scoreColor}" stroke-width="3.2" stroke-dasharray="${data.healthScore}, 100" stroke-linecap="round" fill="none"
                d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" />
            </svg>
            <div style="position: absolute; text-align: center; top: 50%; left: 50%; transform: translate(-50%, -50%); display: flex; flex-direction: column; align-items: center;">
              <span style="font-size: 11px; font-weight: 800; color: var(--geap-text, #1e293b); line-height: 1;">${data.healthScore}</span>
              <span style="font-size: 6px; color: var(--geap-text-secondary, #64748b); text-transform: uppercase; font-weight: 600; margin-top: 1px;">Score</span>
            </div>
          </div>
          <div style="text-align: left; flex: 1;">
            <strong style="font-size: 10px; color: ${data.scoreColor}; display: block; font-weight: 700; text-transform: uppercase; letter-spacing: 0.02em;">${data.scoreLabel}</strong>
            <span style="font-size: 9.5px; color: var(--geap-text-secondary, #64748b); line-height: 1.35; display: block; margin-top: 1px;">${data.scoreDesc}</span>
          </div>
        </div>

        <!-- Asset Allocation bar -->
        <div style="margin-bottom: 14px;">
          <div style="display: flex; justify-content: space-between; font-size: 9px; color: var(--geap-text-secondary, #64748b); font-weight: 600; text-transform: uppercase; margin-bottom: 4px; letter-spacing: 0.03em;">
            <span>Asset Class Allocation</span>
            <span>${data.equitiesPct}% Equities / ${data.cashPct}% Cash</span>
          </div>
          <div style="height: 6px; border-radius: 3px; display: flex; overflow: hidden; background: var(--geap-surface-tertiary, #f1f3f5);">
            <div style="width: ${data.equitiesPct}%; background: linear-gradient(90deg, #7c3aed, #a855f7); height: 100%;"></div>
            <div style="width: ${data.cashPct}%; background: #06b6d4; height: 100%;"></div>
          </div>
          <div style="display: flex; gap: 10px; margin-top: 5px; font-size: 8.5px;">
            <span style="display: flex; align-items: center; gap: 4px; color: var(--geap-text, #1e293b); font-weight: 500;">
              <span style="width: 5px; height: 5px; border-radius: 50%; background: #7c3aed; display: inline-block;"></span>
              Equities (${data.equitiesVal})
            </span>
            <span style="display: flex; align-items: center; gap: 4px; color: var(--geap-text, #1e293b); font-weight: 500;">
              <span style="width: 5px; height: 5px; border-radius: 50%; background: #06b6d4; display: inline-block;"></span>
              Cash (${data.cashVal})
            </span>
          </div>
        </div>

        <!-- Top Holdings & Sector Allocation side-by-side or stacked -->
        <div style="margin-bottom: 14px;">
          <div style="font-size: 9px; color: var(--geap-text-secondary, #64748b); font-weight: 600; text-transform: uppercase; margin-bottom: 6px; letter-spacing: 0.03em;">Portfolio Breakdown</div>
          
          <div style="display: grid; grid-template-columns: 1.15fr 0.85fr; gap: 8px;">
            <!-- Holdings Mini-Table -->
            <div style="border: 1px solid var(--geap-border, #e2e8f0); border-radius: 6px; overflow: hidden; background: var(--geap-surface, #ffffff);">
              <div style="background: var(--geap-surface-secondary, #f8fafc); padding: 4px 6px; font-size: 8px; font-weight: 700; color: var(--geap-text-secondary, #64748b); border-bottom: 1px solid var(--geap-border, #e2e8f0); text-transform: uppercase; letter-spacing: 0.02em;">Top Holdings</div>
              <div style="display: flex; flex-direction: column;">
                ${data.holdingsHtml}
              </div>
            </div>

            <!-- Sectors progress bars -->
            <div style="border: 1px solid var(--geap-border, #e2e8f0); border-radius: 6px; padding: 6px; background: var(--geap-surface, #ffffff); display: flex; flex-direction: column; gap: 6px; justify-content: center;">
              ${data.sectorsHtml}
            </div>
          </div>
        </div>

        <!-- Alerts -->
        ${data.alertsHtml || ''}

        <!-- Actions -->
        <div style="display: flex; gap: 8px; margin-top: 12px;">
          ${data.ctaHtml}
        </div>

        <!-- Status indicator for action -->
        ${data.statusId ? `
          <div id="${data.statusId}" class="rebalance-status-container" style="display: none; align-items: center; gap: 6px; margin-top: 10px; font-size: 10px; color: var(--geap-text-secondary, #64748b);">
            <div class="rebalance-spinner"></div>
            <span>${data.statusText || 'Executing...'}</span>
          </div>
        ` : ''}
      </div>
    `;
  }

  function getAiPortfolioAnalyzerWidgetHtml() {
    const accountId = state.selectedAccountId || 'core';
    
    if (accountId.startsWith("etrade_")) {
      const liveAccounts = BrokerageData.getLiveAccounts ? BrokerageData.getLiveAccounts() : [];
      const acc = liveAccounts.find(a => a.accountIdKey === accountId.replace("etrade_", "")) || liveAccounts[0];
      if (!acc) return `<div class="chat-rebalance-card">Error: E*TRADE Live account data not found.</div>`;
      
      const totalVal = BrokerageData.formatCurrency(acc.totalValue);
      const cashVal = BrokerageData.formatCurrency(acc.cashBalance);
      const equitiesVal = BrokerageData.formatCurrency(acc.totalValue - acc.cashBalance);
      
      const dayChangeVal = acc.dayChange || 0;
      const dayChangePctVal = acc.dayChangePercent || 0;
      const dayChangeStr = `${BrokerageData.formatChange(dayChangeVal)} (${BrokerageData.formatPercent(dayChangePctVal)})`;
      
      const totalValueNum = acc.totalValue || 0;
      const cashValNum = acc.cashBalance || 0;
      const cashPct = totalValueNum > 0 ? Math.round((cashValNum / totalValueNum) * 100) : 0;
      const equitiesPct = 100 - cashPct;
      
      let healthScore = 100;
      let scoreLabel = "Optimized";
      let scoreColor = "#10b981"; // green
      let scoreDesc = "Your E*TRADE account holdings and cash levels are healthy.";
      let alertsHtml = "";
      let ctaHtml = "";
      
      let maxConcentration = 0;
      let concentratedSymbol = "";
      if (acc.holdings && acc.holdings.length > 0) {
        acc.holdings.forEach(h => {
          if (h.weight > maxConcentration) {
            maxConcentration = h.weight;
            concentratedSymbol = h.symbol;
          }
        });
      }
      
      if (maxConcentration > 20) {
        const excess = Math.round(maxConcentration - 15);
        healthScore -= Math.min(40, excess * 2);
        alertsHtml += `
          <div style="background: #fff1f2; border: 1px solid #fecaca; padding: 8px 10px; border-radius: 8px; margin-bottom: 10px; display: flex; gap: 6px; align-items: flex-start; text-align: left;">
            <span style="font-size: 13px;">⚠️</span>
            <div>
              <strong style="color: #be123c; font-size: 10.5px; display: block; font-weight: 700;">Single-Stock Concentration</strong>
              <span style="color: #4f4f4f; font-size: 9.5px; line-height: 1.3; display: block; margin-top: 1px;">
                Your position in <strong>${concentratedSymbol}</strong> accounts for <strong>${maxConcentration.toFixed(1)}%</strong> of your holdings, exceeding the 15% safety threshold.
              </span>
            </div>
          </div>
        `;
        ctaHtml += `
          <button class="rebalance-execute-btn" onclick="triggerChatQuery('suggest rebalancing order to reduce concentration in ${concentratedSymbol}', 'portfolio-analyst')" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
            ✦ Trim ${concentratedSymbol}
          </button>
        `;
      }
      
      if (cashPct > 20) {
        healthScore -= Math.min(25, Math.round((cashPct - 20) * 1.2));
        alertsHtml += `
          <div style="background: #fffbeb; border: 1px solid #fef3c7; padding: 8px 10px; border-radius: 8px; margin-bottom: 10px; display: flex; gap: 6px; align-items: flex-start; text-align: left;">
            <span style="font-size: 13px;">💡</span>
            <div>
              <strong style="color: #b45309; font-size: 10.5px; display: block; font-weight: 700;">Excess Cash Yield Drag</strong>
              <span style="color: #4f4f4f; font-size: 9.5px; line-height: 1.3; display: block; margin-top: 1px;">
                You are holding <strong>${cashPct}%</strong> cash. Sweeping excess cash into high-yield sweep assets (4.50% APY) could generate extra yields.
              </span>
            </div>
          </div>
        `;
        if (!ctaHtml) {
          ctaHtml += `
            <button class="rebalance-execute-btn" onclick="triggerChatQuery('recommend high yield cash sweep', 'portfolio-analyst')" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
              ✦ Sweep Cash
            </button>
          `;
        }
      }
      
      if (healthScore < 65) {
        scoreLabel = "Concentration Risk";
        scoreColor = "#ef4444"; // red
        scoreDesc = "High concentration risk detected. Immediate rebalancing is recommended.";
      } else if (healthScore < 85) {
        scoreLabel = "Moderate Yield Drag";
        scoreColor = "#f59e0b"; // yellow
        scoreDesc = "Sub-optimal cash allocation or minor diversification gaps.";
      }
      
      if (!ctaHtml) {
        ctaHtml = `
          <button class="rebalance-execute-btn" onclick="triggerChatQuery('run active risk analysis', 'portfolio-analyst')" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
            ✦ Run Active Risk Analysis
          </button>
        `;
      }
      
      let holdingsHtml = "";
      if (acc.holdings && acc.holdings.length > 0) {
        holdingsHtml = acc.holdings.slice(0, 3).map(h => `
          <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed var(--geap-border, #e2e8f0);">
            <div>
              <strong style="color: var(--geap-text, #1e293b);">${h.symbol}</strong>
              <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">${h.shares} shrs</span>
            </div>
            <div style="text-align: right;">
              <span style="color: var(--geap-text, #1e293b); font-weight: 500;">${BrokerageData.formatCurrency(h.marketValue)}</span>
              <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">${h.weight.toFixed(1)}%</span>
            </div>
          </div>
        `).join('');
      } else {
        holdingsHtml = `<div style="padding: 10px; text-align: center; color: var(--geap-text-secondary, #64748b); font-size: 9.5px;">No active holdings.</div>`;
      }
      
      let sectorsHtml = "";
      const sectors = acc.sectorAllocation || {};
      if (Object.keys(sectors).length > 0) {
        sectorsHtml = Object.entries(sectors)
          .sort((a, b) => b[1].weight - a[1].weight)
          .slice(0, 3)
          .map(([sectorName, data]) => {
            const weight = data.weight;
            const color = data.color || '#64748b';
            return `
              <div style="text-align: left;">
                <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
                  <span style="font-weight: 500; color: var(--geap-text, #1e293b); text-overflow: ellipsis; overflow: hidden; white-space: nowrap; max-width: 60px;">${sectorName}</span>
                  <strong style="color: var(--geap-text, #1e293b);">${weight.toFixed(1)}%</strong>
                </div>
                <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
                  <div style="width: ${weight}%; height: 100%; background: ${color}; border-radius: 2px;"></div>
                </div>
              </div>
            `;
          }).join('');
      } else {
        sectorsHtml = `<div style="text-align: center; color: var(--geap-text-secondary, #64748b); font-size: 8.5px;">No sector exposure.</div>`;
      }
      
      return renderVisualReview({
        widgetId: "analyzer-widget-etrade",
        title: "✦ Live E*TRADE Diagnostics",
        badge: "Live Mode",
        badgeStyle: "background: #ecfdf5; color: #059669; border: 1px solid #a7f3d0;",
        netAssets: totalVal,
        dayChange: dayChangeStr,
        dayChangeColor: acc.dayChange >= 0 ? 'var(--positive)' : 'var(--negative)',
        healthScore: healthScore,
        scoreColor: scoreColor,
        scoreLabel: scoreLabel,
        scoreDesc: scoreDesc,
        equitiesPct: equitiesPct,
        cashPct: cashPct,
        equitiesVal: equitiesVal,
        cashVal: cashVal,
        holdingsHtml: holdingsHtml,
        sectorsHtml: sectorsHtml,
        alertsHtml: alertsHtml,
        ctaHtml: ctaHtml,
        statusId: "analyzer-etrade-status",
        statusText: "Analyzing E*TRADE positions..."
      });
    }
    
    if (accountId === "main") {
      const acc = mockData.accounts.find(a => a.id === 'main');
      const netAssets = acc ? acc.netValue : "$18,427.55";
      const dayChange = acc ? acc.dayGain : "-$82.14 (-0.44%)";
      const dayChangeTone = acc ? acc.dayGainTone : "negative";
      
      let holdingsHtml = `
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed var(--geap-border, #e2e8f0);">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">BAC</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">148 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$8,008.28</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">43.5%</span>
          </div>
        </div>
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed var(--geap-border, #e2e8f0);">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">KO</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">77 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$5,916.68</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">32.0%</span>
          </div>
        </div>
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center;">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">BND</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">38 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$2,741.70</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">15.0%</span>
          </div>
        </div>
      `;
      
      let sectorsHtml = `
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Financials</span>
            <strong style="color: var(--geap-text, #1e293b);">43.5%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 43.5%; height: 100%; background: #3b82f6; border-radius: 2px;"></div>
          </div>
        </div>
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Consumer</span>
            <strong style="color: var(--geap-text, #1e293b);">32.0%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 32%; height: 100%; background: #ec4899; border-radius: 2px;"></div>
          </div>
        </div>
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Bonds</span>
            <strong style="color: var(--geap-text, #1e293b);">15.0%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 15%; height: 100%; background: #10b981; border-radius: 2px;"></div>
          </div>
        </div>
      `;
      
      let alertsHtml = `
        <div style="background: #fff1f2; border: 1px solid #fecaca; padding: 8px 10px; border-radius: 8px; margin-bottom: 10px; display: flex; gap: 6px; align-items: flex-start; text-align: left;">
          <span style="font-size: 13px;">⚠️</span>
          <div>
            <strong style="color: #be123c; font-size: 10.5px; display: block; font-weight: 700;">Single-Stock Concentration</strong>
            <span style="color: #4f4f4f; font-size: 9.5px; line-height: 1.3; display: block; margin-top: 1px;">
              Your position in <strong>BAC</strong> represents <strong>43.5%</strong> of holdings. This exceeds the 15% threshold, exposing you to high volatility.
            </span>
          </div>
        </div>
      `;
      
      let ctaHtml = `
        <button class="rebalance-execute-btn" onclick="Chat.executeConcentrationTrim(this)" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
          ✦ Trim BAC Concentration
        </button>
        <button class="rebalance-execute-btn" onclick="routeTo('/accounts/portfolios')" style="flex: 1; background: transparent; border-color: #6B2D9B; color: #6B2D9B; font-size: 0.8rem; padding: 8px 12px;">
          View Sector Alloc
        </button>
      `;

      return renderVisualReview({
        widgetId: "analyzer-widget-main",
        title: "✦ Portfolio Optimization Dashboard",
        badge: "Ind Brokerage",
        badgeStyle: "background: #f5f3ff; color: #7c3aed; border: 1px solid #ddd6fe;",
        netAssets: netAssets,
        dayChange: dayChange,
        dayChangeColor: dayChangeTone === 'positive' ? 'var(--positive)' : 'var(--negative)',
        healthScore: 54,
        scoreColor: "#ef4444",
        scoreLabel: "Poor Diversification",
        scoreDesc: "Overconcentrated position in a single financial sector asset.",
        equitiesPct: 85,
        cashPct: 15,
        equitiesVal: "$15,663.85",
        cashVal: "$2,763.70",
        holdingsHtml: holdingsHtml,
        sectorsHtml: sectorsHtml,
        alertsHtml: alertsHtml,
        ctaHtml: ctaHtml,
        statusId: "analyzer-main-status",
        statusText: "Staging order: Sell 100 BAC, reallocate to BND..."
      });
    }
    
    if (accountId === "core") {
      const acc = mockData.accounts.find(a => a.id === 'core');
      const netAssets = acc ? acc.netValue : "$12,734.88";
      const dayChange = acc ? acc.dayGain : "$56.41 (0.44%)";
      const dayChangeTone = acc ? acc.dayGainTone : "positive";
      
      let holdingsHtml = `
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed var(--geap-border, #e2e8f0);">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">QQQ</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">12 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$6,200.00</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">48.7%</span>
          </div>
        </div>
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed var(--geap-border, #e2e8f0);">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">RIVN</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">226 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$4,100.00</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">32.2%</span>
          </div>
        </div>
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center;">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">MBLY</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">231 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$2,434.88</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">19.1%</span>
          </div>
        </div>
      `;
      
      let sectorsHtml = `
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Tech Index</span>
            <strong style="color: var(--geap-text, #1e293b);">48.7%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 48.7%; height: 100%; background: #7c3aed; border-radius: 2px;"></div>
          </div>
        </div>
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Automotive</span>
            <strong style="color: var(--geap-text, #1e293b);">32.2%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 32.2%; height: 100%; background: #3b82f6; border-radius: 2px;"></div>
          </div>
        </div>
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Hardware</span>
            <strong style="color: var(--geap-text, #1e293b);">19.1%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 19.1%; height: 100%; background: #10b981; border-radius: 2px;"></div>
          </div>
        </div>
      `;
      
      let alertsHtml = `
        <div style="background: #eef2ff; border: 1px solid #e0e7ff; padding: 8px 10px; border-radius: 8px; margin-bottom: 10px; display: flex; gap: 6px; align-items: flex-start; text-align: left;">
          <span style="font-size: 13px;">📊</span>
          <div>
            <strong style="color: #312e81; font-size: 10.5px; display: block; font-weight: 700;">Dollar-Cost Averaging</strong>
            <span style="color: #4f4f4f; font-size: 9.5px; line-height: 1.3; display: block; margin-top: 1px;">
              To maximize the compounding potential of this smart beta allocation, we suggest setting up a recurring <strong>$250 monthly Auto-Investment</strong>.
            </span>
          </div>
        </div>
      `;
      
      let ctaHtml = `
        <button class="rebalance-execute-btn" onclick="Chat.executeAutoInvest(this)" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
          ✦ Enable Auto-Investing
        </button>
        <button class="rebalance-execute-btn" onclick="state.portfolioTab = 'risk'; render(); triggerChatQuery('run active risk analysis', 'portfolio-analyst');" style="flex: 1; background: transparent; border-color: #6B2D9B; color: #6B2D9B; font-size: 0.8rem; padding: 8px 12px;">
          Run Risk Diagnostic
        </button>
      `;

      return renderVisualReview({
        widgetId: "analyzer-widget-core",
        title: "✦ Smart Beta Strategy Optimization",
        badge: "Core Portfolios",
        badgeStyle: "background: #f5f3ff; color: #7c3aed; border: 1px solid #ddd6fe;",
        netAssets: netAssets,
        dayChange: dayChange,
        dayChangeColor: dayChangeTone === 'positive' ? 'var(--positive)' : 'var(--negative)',
        healthScore: 88,
        scoreColor: "#10b981",
        scoreLabel: "Optimized Allocation",
        scoreDesc: "Active Smart Beta portfolio outperforming baseline benchmarks.",
        equitiesPct: 98,
        cashPct: 2,
        equitiesVal: "$12,492.12",
        cashVal: "$242.76",
        holdingsHtml: holdingsHtml,
        sectorsHtml: sectorsHtml,
        alertsHtml: alertsHtml,
        ctaHtml: ctaHtml,
        statusId: "analyzer-core-status",
        statusText: "Configuring recurring transfer plan..."
      });
    }
    
    if (accountId === "elcie") {
      const acc = mockData.accounts.find(a => a.id === 'elcie');
      const netAssets = acc ? acc.netValue : "$3,608.94";
      const dayChange = acc ? acc.dayGain : "-$11.08 (-0.29%)";
      const dayChangeTone = acc ? acc.dayGainTone : "negative";
      
      let holdingsHtml = `
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed var(--geap-border, #e2e8f0);">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">AAPL</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">9 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$1,800.00</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">49.9%</span>
          </div>
        </div>
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center;">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">BND</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">21 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$1,490.49</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">41.3%</span>
          </div>
        </div>
      `;
      
      let sectorsHtml = `
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Technology</span>
            <strong style="color: var(--geap-text, #1e293b);">49.9%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 49.9%; height: 100%; background: #7c3aed; border-radius: 2px;"></div>
          </div>
        </div>
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Bonds</span>
            <strong style="color: var(--geap-text, #1e293b);">41.3%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 41.3%; height: 100%; background: #10b981; border-radius: 2px;"></div>
          </div>
        </div>
      `;
      
      let alertsHtml = `
        <div style="background: #eef2ff; border: 1px solid #e0e7ff; padding: 8px 10px; border-radius: 8px; margin-bottom: 10px; display: flex; gap: 6px; align-items: flex-start; text-align: left;">
          <span style="font-size: 13px;">🎓</span>
          <div>
            <strong style="color: #312e81; font-size: 10.5px; display: block; font-weight: 700;">Glide Path Transition</strong>
            <span style="color: #4f4f4f; font-size: 9.5px; line-height: 1.3; display: block; margin-top: 1px;">
              As the beneficiary approaches college age, we recommend transitioning to an age-based targets fund to shield savings.
            </span>
          </div>
        </div>
      `;
      
      let ctaHtml = `
        <button class="rebalance-execute-btn" onclick="Chat.executeGlidePathTransition(this)" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
          ✦ Transition to Glide Path
        </button>
        <button class="rebalance-execute-btn" onclick="triggerChatQuery('tax loss harvesting analysis', 'portfolio-analyst')" style="flex: 1; background: transparent; border-color: #6B2D9B; color: #6B2D9B; font-size: 0.8rem; padding: 8px 12px;">
          Harvest Losses
        </button>
      `;

      return renderVisualReview({
        widgetId: "analyzer-widget-elcie",
        title: "✦ UTMA/UGMA Education Optimization",
        badge: "Custodial UGMA",
        badgeStyle: "background: #f5f3ff; color: #7c3aed; border: 1px solid #ddd6fe;",
        netAssets: netAssets,
        dayChange: dayChange,
        dayChangeColor: dayChangeTone === 'positive' ? 'var(--positive)' : 'var(--negative)',
        healthScore: 82,
        scoreColor: "#10b981",
        scoreLabel: "Balanced Target",
        scoreDesc: "Age-appropriate asset distribution with moderate loss harvest opportunities.",
        equitiesPct: 75,
        cashPct: 25,
        equitiesVal: "$2,708.94",
        cashVal: "$900.00",
        holdingsHtml: holdingsHtml,
        sectorsHtml: sectorsHtml,
        alertsHtml: alertsHtml,
        ctaHtml: ctaHtml,
        statusId: "analyzer-elcie-status",
        statusText: "Transitioning allocation model..."
      });
    }
    
    if (accountId === "checking") {
      const acc = mockData.accounts.find(a => a.id === 'checking');
      const netAssets = acc ? acc.netValue : "$6,941.26";
      
      let holdingsHtml = `
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center;">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">Liquid Cash</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">Private Bank</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">${netAssets}</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">100.0%</span>
          </div>
        </div>
      `;
      
      let sectorsHtml = `
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Bank Sweep</span>
            <strong style="color: var(--geap-text, #1e293b);">100.0%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 100%; height: 100%; background: #06b6d4; border-radius: 2px;"></div>
          </div>
        </div>
      `;
      
      let alertsHtml = `
        <div style="background: #fffbeb; border: 1px solid #fef3c7; padding: 8px 10px; border-radius: 8px; margin-bottom: 10px; display: flex; gap: 6px; align-items: flex-start; text-align: left;">
          <span style="font-size: 13px;">💡</span>
          <div>
            <strong style="color: #b45309; font-size: 10.5px; display: block; font-weight: 700;">Sub-Optimal Cash Yield</strong>
            <span style="color: #4f4f4f; font-size: 9.5px; line-height: 1.3; display: block; margin-top: 1px;">
              You are holding ${netAssets} in cash earning near-zero (0.05%) interest. Sweeping <strong>$4,941.26</strong> to high-yield sweep yields <strong>4.50% APY</strong> (+$222/yr).
            </span>
          </div>
        </div>
      `;
      
      let ctaHtml = `
        <button class="rebalance-execute-btn" onclick="Chat.executeYieldSweep(this)" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
          ✦ Sweep Excess Cash ($4,941)
        </button>
        <button class="rebalance-execute-btn" onclick="routeTo('/pay-transfer')" style="flex: 1; background: transparent; border-color: #6B2D9B; color: #6B2D9B; font-size: 0.8rem; padding: 8px 12px;">
          Transfer to Brokerage
        </button>
      `;

      return renderVisualReview({
        widgetId: "analyzer-widget-checking",
        title: "✦ Cash Yield Optimization Analysis",
        badge: "Private Checking",
        badgeStyle: "background: #f5f3ff; color: #7c3aed; border: 1px solid #ddd6fe;",
        netAssets: netAssets,
        dayChange: "+$0.00 (0.00%)",
        dayChangeColor: "var(--geap-text-secondary)",
        healthScore: 45,
        scoreColor: "#ef4444",
        scoreLabel: "Sub-Optimal Yield",
        scoreDesc: "Idle liquid cash represents a drag on your net yield strategy.",
        equitiesPct: 0,
        cashPct: 100,
        equitiesVal: "$0.00",
        cashVal: netAssets,
        holdingsHtml: holdingsHtml,
        sectorsHtml: sectorsHtml,
        alertsHtml: alertsHtml,
        ctaHtml: ctaHtml,
        statusId: "analyzer-checking-status",
        statusText: "Moving funds to High-Yield Cash sweep..."
      });
    }
    
    if (accountId === "individual") {
      const acc = mockData.accounts.find(a => a.id === 'individual');
      const netAssets = acc ? acc.netValue : "$4,285.17";
      const dayChange = acc ? acc.dayGain : "$18.95 (0.44%)";
      const dayChangeTone = acc ? acc.dayGainTone : "positive";
      
      let holdingsHtml = `
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed var(--geap-border, #e2e8f0);">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">MSFT</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">5 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$2,200.00</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">51.3%</span>
          </div>
        </div>
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center;">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">VTI</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">5 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$1,321.05</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">30.8%</span>
          </div>
        </div>
      `;
      
      let sectorsHtml = `
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Technology</span>
            <strong style="color: var(--geap-text, #1e293b);">51.3%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 51.3%; height: 100%; background: #7c3aed; border-radius: 2px;"></div>
          </div>
        </div>
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Broad Market</span>
            <strong style="color: var(--geap-text, #1e293b);">30.8%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 30.8%; height: 100%; background: #3b82f6; border-radius: 2px;"></div>
          </div>
        </div>
      `;
      
      let alertsHtml = `
        <div style="background: #eef2ff; border: 1px solid #e0e7ff; padding: 8px 10px; border-radius: 8px; margin-bottom: 10px; display: flex; gap: 6px; align-items: flex-start; text-align: left;">
          <span style="font-size: 13px;">💡</span>
          <div>
            <strong style="color: #312e81; font-size: 10.5px; display: block; font-weight: 700;">Diversification Check</strong>
            <span style="color: #4f4f4f; font-size: 9.5px; line-height: 1.3; display: block; margin-top: 1px;">
              Review combined allocations across all linked accounts to check for tech overlap.
            </span>
          </div>
        </div>
      `;
      
      let ctaHtml = `
        <button class="rebalance-execute-btn" onclick="routeTo('/accounts/portfolios')" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
          ✦ Analyze Sector Allocation
        </button>
      `;

      return renderVisualReview({
        widgetId: "analyzer-widget-individual",
        title: "✦ Asset Allocation Optimizer",
        badge: "Individual Brokerage",
        badgeStyle: "background: #f5f3ff; color: #7c3aed; border: 1px solid #ddd6fe;",
        netAssets: netAssets,
        dayChange: dayChange,
        dayChangeColor: dayChangeTone === 'positive' ? 'var(--positive)' : 'var(--negative)',
        healthScore: 85,
        scoreColor: "#10b981",
        scoreLabel: "Well Diversified",
        scoreDesc: "Assets and sectors are properly distributed across multiple categories.",
        equitiesPct: 82,
        cashPct: 18,
        equitiesVal: "$3,521.05",
        cashVal: "$764.12",
        holdingsHtml: holdingsHtml,
        sectorsHtml: sectorsHtml,
        alertsHtml: alertsHtml,
        ctaHtml: ctaHtml
      });
    }
    
    if (accountId === "stock") {
      const acc = mockData.accounts.find(a => a.id === 'stock');
      const netAssets = acc ? acc.netValue : "$5,914.92";
      const dayChange = acc ? acc.dayGain : "$136.78 (2.37%)";
      const dayChangeTone = acc ? acc.dayGainTone : "positive";
      
      let holdingsHtml = `
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed var(--geap-border, #e2e8f0);">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">MSFT RSU</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">11 shrs</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$5,114.92</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">86.5%</span>
          </div>
        </div>
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center;">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">MSFT Opt</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">3 contracts</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$800.00</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">13.5%</span>
          </div>
        </div>
      `;
      
      let sectorsHtml = `
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Company Stock</span>
            <strong style="color: var(--geap-text, #1e293b);">86.5%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 86.5%; height: 100%; background: #7c3aed; border-radius: 2px;"></div>
          </div>
        </div>
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Derivatives</span>
            <strong style="color: var(--geap-text, #1e293b);">13.5%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 13.5%; height: 100%; background: #ec4899; border-radius: 2px;"></div>
          </div>
        </div>
      `;
      
      let alertsHtml = `
        <div style="background: #fffbeb; border: 1px solid #fef3c7; padding: 8px 10px; border-radius: 8px; margin-bottom: 10px; display: flex; gap: 6px; align-items: flex-start; text-align: left;">
          <span style="font-size: 13px;">🛡️</span>
          <div>
            <strong style="color: #b45309; font-size: 10.5px; display: block; font-weight: 700;">Vesting Concentration Hedging</strong>
            <span style="color: #4f4f4f; font-size: 9.5px; line-height: 1.3; display: block; margin-top: 1px;">
              Your upcoming vestings are heavily tied to company equity. Consider options collars to hedge.
            </span>
          </div>
        </div>
      `;
      
      let ctaHtml = `
        <button class="rebalance-execute-btn" onclick="Chat.viewVestingSchedule(this)" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
          ✦ View Vesting Schedule
        </button>
        <button class="rebalance-execute-btn" onclick="triggerChatQuery('explain option collars', 'market-research')" style="flex: 1; background: transparent; border-color: #6B2D9B; color: #6B2D9B; font-size: 0.8rem; padding: 8px 12px;">
          Option Collars
        </button>
      `;

      return renderVisualReview({
        widgetId: "analyzer-widget-stock",
        title: "✦ Equity Award Vesting & Tax Strategy",
        badge: "Stock Plan",
        badgeStyle: "background: #f5f3ff; color: #7c3aed; border: 1px solid #ddd6fe;",
        netAssets: netAssets,
        dayChange: dayChange,
        dayChangeColor: dayChangeTone === 'positive' ? 'var(--positive)' : 'var(--negative)',
        healthScore: 62,
        scoreColor: "#f59e0b",
        scoreLabel: "Vesting Concentration",
        scoreDesc: "High exposure to single employer stock awards with upcoming vestings.",
        equitiesPct: 100,
        cashPct: 0,
        equitiesVal: netAssets,
        cashVal: "$0.00",
        holdingsHtml: holdingsHtml,
        sectorsHtml: sectorsHtml,
        alertsHtml: alertsHtml,
        ctaHtml: ctaHtml,
        statusId: "analyzer-stock-status",
        statusText: "Loading vesting calendar details..."
      });
    }
    
    if (accountId === "morgan_stanley") {
      const account = mockData.morganStanleyAccount;
      const netAssets = account ? account.assets : "$8,104.31";
      const dayChange = account ? `${account.dayGain} (${account.dayGainPct})` : "+$34.20 (+0.42%)";
      
      let holdingsHtml = `
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed var(--geap-border, #e2e8f0);">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">MS Growth</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">Private Fund</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$4,200.00</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">51.8%</span>
          </div>
        </div>
        <div style="padding: 5px 6px; font-size: 9.5px; display: flex; justify-content: space-between; align-items: center;">
          <div>
            <strong style="color: var(--geap-text, #1e293b);">Fixed Inc</strong>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); margin-left: 4px;">MS Bonds</span>
          </div>
          <div style="text-align: right;">
            <span style="color: var(--geap-text, #1e293b); font-weight: 500;">$2,283.45</span>
            <span style="font-size: 8px; color: var(--geap-text-secondary, #64748b); display: block;">28.2%</span>
          </div>
        </div>
      `;
      
      let sectorsHtml = `
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Growth Funds</span>
            <strong style="color: var(--geap-text, #1e293b);">51.8%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 51.8%; height: 100%; background: #7c3aed; border-radius: 2px;"></div>
          </div>
        </div>
        <div style="text-align: left;">
          <div style="display: flex; justify-content: space-between; font-size: 8.5px; margin-bottom: 1px;">
            <span style="font-weight: 500; color: var(--geap-text, #1e293b);">Fixed Income</span>
            <strong style="color: var(--geap-text, #1e293b);">28.2%</strong>
          </div>
          <div style="height: 3.5px; border-radius: 2px; background: var(--geap-surface-tertiary, #f1f3f5); overflow: hidden;">
            <div style="width: 28.2%; height: 100%; background: #10b981; border-radius: 2px;"></div>
          </div>
        </div>
      `;
      
      let alertsHtml = `
        <div style="background: #eef2ff; border: 1px solid #e0e7ff; padding: 8px 10px; border-radius: 8px; margin-bottom: 10px; display: flex; gap: 6px; align-items: flex-start; text-align: left;">
          <span style="font-size: 13px;">📞</span>
          <div>
            <strong style="color: #312e81; font-size: 10.5px; display: block; font-weight: 700;">Wealth Review Scheduled</strong>
            <span style="color: #4f4f4f; font-size: 9.5px; line-height: 1.3; display: block; margin-top: 1px;">
              Annual private wealth reviews help align long-term tax and inheritance distributions.
            </span>
          </div>
        </div>
      `;
      
      let ctaHtml = `
        <button class="rebalance-execute-btn" onclick="alert('Advisory Call requested. Your private advisor will contact you within 24 hours.')" style="flex: 1; font-size: 0.8rem; padding: 8px 12px; background: var(--purple-bright, #6B2D9B);">
          ✦ Schedule Advisor Call
        </button>
        <button class="rebalance-execute-btn" onclick="routeTo('/planning')" style="flex: 1; background: transparent; border-color: #6B2D9B; color: #6B2D9B; font-size: 0.8rem; padding: 8px 12px;">
          View Retirement Plan
        </button>
      `;

      return renderVisualReview({
        widgetId: "analyzer-widget-ms",
        title: "✦ Morgan Stanley Private Wealth Analysis",
        badge: "Private Wealth",
        badgeStyle: "background: #f5f3ff; color: #7c3aed; border: 1px solid #ddd6fe;",
        netAssets: netAssets,
        dayChange: dayChange,
        dayChangeColor: "var(--positive)",
        healthScore: 80,
        scoreColor: "#10b981",
        scoreLabel: "Conservative Balanced",
        scoreDesc: "Assets are appropriately positioned for capital preservation and security.",
        equitiesPct: 80,
        cashPct: 20,
        equitiesVal: "$6,483.45",
        cashVal: "$1,620.86",
        holdingsHtml: holdingsHtml,
        sectorsHtml: sectorsHtml,
        alertsHtml: alertsHtml,
        ctaHtml: ctaHtml
      });
    }
    
    return `<div class="chat-rebalance-card">Error: Unrecognized account context "${accountId}".</div>`;
  }

  function executeYieldSweep(btn) {
    const statusContainer = document.getElementById('analyzer-checking-status');
    if (!statusContainer) return;
    
    btn.style.display = 'none';
    statusContainer.style.display = 'flex';
    
    setTimeout(() => {
      const sweepAmt = 4941.26;
      if (typeof mockData !== 'undefined' && Array.isArray(mockData.accounts)) {
        const checking = mockData.accounts.find(a => a.id === 'checking');
        if (checking) {
          checking.netValue = "$2,000.00";
          checking.available = "$2,000.00";
          checking.totalBalance = "$2,000.00";
        }
        
        // Add yield assets into checking or create a new row or just keep totals updated.
        // Let's also update the total asset figures if we want, but checking netValue change
        // is visible on next dashboard redraw. Let's make sure total assets are not reduced
        // since it is just swept within Morgan Stanley.
      }
      
      const card = btn.closest('.chat-rebalance-card');
      if (card) {
        card.innerHTML = `
          <div class="rebalance-success-state">
            <div class="success-icon-wrapper" style="background: #ecfdf5; color: #059669; border: none; width: 40px; height: 40px; border-radius: 50%; display: flex; align-items: center; justify-content: center; margin: 0 auto 12px auto;">
              <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 52 52" style="width: 24px; height: 24px; fill: none; stroke: currentColor; stroke-width: 4;">
                <circle cx="26" cy="26" r="25" fill="none" />
                <path d="M14.1 27.2l7.1 7.2 16.7-16.8" />
              </svg>
            </div>
            <h3>Yield Sweep Activated</h3>
            <p class="success-desc">Transferred $4,941.26 from Checking into High-Yield Savings (compounding at 4.50% APY).</p>
            <div class="success-details">
              <div><strong>Reference ID:</strong> SWP-820941-APY</div>
              <div><strong>Status:</strong> Active & Compounding</div>
            </div>
          </div>
        `;
      }
      
      if (typeof render === 'function') {
        render();
      }
      
      if (typeof BrokerageData !== 'undefined' && typeof BrokerageData.logAgentActivity === 'function') {
        BrokerageData.logAgentActivity({
          agent: 'Portfolio Analyst',
          agentId: 'portfolio-analyst',
          query: 'Yield sweep transaction of $4,941.26 from Checking (-8152) to High-Yield Cash Sweep.',
          status: 'success',
          latency: '0.8',
        });
      }
    }, 2000);
  }

  function executeConcentrationTrim(btn) {
    const statusContainer = document.getElementById('analyzer-main-status');
    if (!statusContainer) return;
    
    btn.style.display = 'none';
    statusContainer.style.display = 'flex';
    
    setTimeout(() => {
      const card = btn.closest('.chat-rebalance-card');
      if (card) {
        card.innerHTML = `
          <div class="rebalance-success-state">
            <div class="success-icon-wrapper" style="background: #e0f2fe; color: #0284c7; border: none; width: 40px; height: 40px; border-radius: 50%; display: flex; align-items: center; justify-content: center; margin: 0 auto 12px auto;">
              <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 52 52" style="width: 24px; height: 24px; fill: none; stroke: currentColor; stroke-width: 4;">
                <circle cx="26" cy="26" r="25" fill="none" />
                <path d="M14.1 27.2l7.1 7.2 16.7-16.8" />
              </svg>
            </div>
            <h3>BAC Rebalance Staged</h3>
            <p class="success-desc">Sell 100 shares of BAC at market price (~$5,411) and buy 75 shares of BND. Staged in your review queue.</p>
            <div class="success-details">
              <div><strong>Reference ID:</strong> STG-094186-BAC</div>
              <div><strong>Status:</strong> Staged for Customer Review</div>
            </div>
          </div>
        `;
      }
      
      if (typeof BrokerageData !== 'undefined' && typeof BrokerageData.logAgentActivity === 'function') {
        BrokerageData.logAgentActivity({
          agent: 'Portfolio Analyst',
          agentId: 'portfolio-analyst',
          query: 'Staged trim concentration order: Sell 100 shares BAC, buy 75 shares BND.',
          status: 'success',
          latency: '1.2',
        });
      }
    }, 2000);
  }

  function executeAutoInvest(btn) {
    const statusContainer = document.getElementById('analyzer-core-status');
    if (!statusContainer) return;
    
    btn.style.display = 'none';
    statusContainer.style.display = 'flex';
    
    setTimeout(() => {
      const card = btn.closest('.chat-rebalance-card');
      if (card) {
        card.innerHTML = `
          <div class="rebalance-success-state">
            <div class="success-icon-wrapper" style="background: #ecfdf5; color: #059669; border: none; width: 40px; height: 40px; border-radius: 50%; display: flex; align-items: center; justify-content: center; margin: 0 auto 12px auto;">
              <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 52 52" style="width: 24px; height: 24px; fill: none; stroke: currentColor; stroke-width: 4;">
                <circle cx="26" cy="26" r="25" fill="none" />
                <path d="M14.1 27.2l7.1 7.2 16.7-16.8" />
              </svg>
            </div>
            <h3>Auto-Investing Enabled</h3>
            <p class="success-desc">Configured recurring monthly transfer of $250.00 from checking to your CORE Smart Beta Portfolio on the 1st of every month.</p>
            <div class="success-details">
              <div><strong>Reference ID:</strong> AUT-094186-DCA</div>
              <div><strong>Status:</strong> Active (First Transfer July 1st)</div>
            </div>
          </div>
        `;
      }
      
      if (typeof BrokerageData !== 'undefined' && typeof BrokerageData.logAgentActivity === 'function') {
        BrokerageData.logAgentActivity({
          agent: 'Portfolio Analyst',
          agentId: 'portfolio-analyst',
          query: 'Configured monthly Auto-Investment plan ($250/mo) for CORE portfolio.',
          status: 'success',
          latency: '0.9',
         });
      }
    }, 2000);
  }

  function executeGlidePathTransition(btn) {
    const statusContainer = document.getElementById('analyzer-elcie-status');
    if (!statusContainer) return;
    
    btn.style.display = 'none';
    statusContainer.style.display = 'flex';
    
    setTimeout(() => {
      const card = btn.closest('.chat-rebalance-card');
      if (card) {
        card.innerHTML = `
          <div class="rebalance-success-state">
            <div class="success-icon-wrapper" style="background: #e0f2fe; color: #0284c7; border: none; width: 40px; height: 40px; border-radius: 50%; display: flex; align-items: center; justify-content: center; margin: 0 auto 12px auto;">
              <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 52 52" style="width: 24px; height: 24px; fill: none; stroke: currentColor; stroke-width: 4;">
                <circle cx="26" cy="26" r="25" fill="none" />
                <path d="M14.1 27.2l7.1 7.2 16.7-16.8" />
              </svg>
            </div>
            <h3>Target Glide Path Activated</h3>
            <p class="success-desc">Staged a model allocation shift to Target-Date Custodial Glide Path (Gradual automated stock-to-bond reallocation).</p>
            <div class="success-details">
              <div><strong>Reference ID:</strong> GLD-094186-UTMA</div>
              <div><strong>Status:</strong> Staged for Custodial Approval</div>
            </div>
          </div>
        `;
      }
      
      if (typeof BrokerageData !== 'undefined' && typeof BrokerageData.logAgentActivity === 'function') {
        BrokerageData.logAgentActivity({
          agent: 'Portfolio Analyst',
          agentId: 'portfolio-analyst',
          query: 'Staged model transition to age-based Custodial Glide Path for UTMA/UGMA.',
          status: 'success',
          latency: '1.1',
        });
      }
    }, 2000);
  }

  function viewVestingSchedule(btn) {
    const statusContainer = document.getElementById('analyzer-stock-status');
    if (!statusContainer) return;
    
    btn.style.display = 'none';
    statusContainer.style.display = 'flex';
    
    setTimeout(() => {
      const card = btn.closest('.chat-rebalance-card');
      if (card) {
        card.innerHTML = `
          <div class="rebalance-success-state" style="text-align: left;">
            <h3 style="text-align: center; margin-bottom: 12px; margin-top: 0;">🗓️ Upcoming Vesting Calendar</h3>
            <div style="font-size: 11px; color: #475569; display: flex; flex-direction: column; gap: 8px; border-top: 1px solid #e2e8f0; padding-top: 8px;">
              <div style="display: flex; justify-content: space-between;">
                <span><strong>July 15, 2026</strong></span>
                <span>50 RSU shares Vesting</span>
                <span>Est. Value: $1,850</span>
              </div>
              <div style="display: flex; justify-content: space-between; border-top: 1px dashed #e2e8f0; padding-top: 4px;">
                <span><strong>Oct 15, 2026</strong></span>
                <span>50 RSU shares Vesting</span>
                <span>Est. Value: $1,920</span>
              </div>
              <div style="display: flex; justify-content: space-between; border-top: 1px dashed #e2e8f0; padding-top: 4px;">
                <span><strong>Jan 15, 2027</strong></span>
                <span>120 Option contracts Vest</span>
                <span>Est. Value: $3,100</span>
              </div>
            </div>
            <div style="font-size: 10px; color: #64748b; margin-top: 10px; text-align: center;">
              *RSUs are configured with "Sell-to-Cover" for tax withholding.
            </div>
          </div>
        `;
      }
      
      if (typeof BrokerageData !== 'undefined' && typeof BrokerageData.logAgentActivity === 'function') {
        BrokerageData.logAgentActivity({
          agent: 'Support Agent',
          agentId: 'customer-support',
          query: 'Queried upcoming Stock Plan vesting schedule details.',
          status: 'success',
          latency: '0.7',
        });
      }
    }, 1500);
  }

```


## Appendix E — Gemini tool declarations, fallback text & Fabric sample queries




**`public/js/gemini.js` lines 20–140 (verbatim):**

```js
  const toolDeclarations = [
    {
      name: 'getPortfolioHoldings',
      description: 'Get the full list of portfolio holdings with symbol, shares, price, market value, return, and sector for each position.',
      parameters: {
        type: 'object',
        properties: {},
        required: [],
      },
    },
    {
      name: 'getAccountSummary',
      description: 'Get a summary of the brokerage account including total value, day change, total return, cash balance, top holdings, and sector breakdown.',
      parameters: {
        type: 'object',
        properties: {},
        required: [],
      },
    },
    {
      name: 'getQuote',
      description: 'Get a real-time stock quote for a given ticker symbol, including price, change, volume, market cap, P/E ratio, and 52-week range.',
      parameters: {
        type: 'object',
        properties: {
          symbol: {
            type: 'string',
            description: 'The stock ticker symbol (e.g., AAPL, GOOGL, MSFT)',
          },
        },
        required: ['symbol'],
      },
    },
    {
      name: 'getSectorAllocation',
      description: 'Get the portfolio sector allocation breakdown showing each sector name, weight percentage, and total dollar value.',
      parameters: {
        type: 'object',
        properties: {},
        required: [],
      },
    },
    {
      name: 'getMarketSummary',
      description: 'Get current market index values (S&P 500, NASDAQ, etc.) and check if the NYSE/NASDAQ stock exchanges are currently open or closed (trading hours, pre-market, after-hours).',
      parameters: {
        type: 'object',
        properties: {},
        required: [],
      },
    },
    {
      name: 'getFAQ',
      description: 'Get frequently asked questions and answers about the brokerage platform, including trading fees, transfers, margin, tax documents, and order types.',
      parameters: {
        type: 'object',
        properties: {},
        required: [],
      },
    },
    {
      name: 'navigateToPage',
      description: 'Navigate the platform UI to a specific page path (e.g., "/accounts", "/accounts/portfolios", "/trading", "/accounts/watch-lists", "/accounts/activity", "/agent-studio", "/agent-fabric").',
      parameters: {
        type: 'object',
        properties: {
          path: {
            type: 'string',
            description: 'The destination page path to route the user to.'
          }
        },
        required: ['path'],
      },
    },
    {
      name: 'selectAccount',
      description: 'Select a specific account context in the UI (e.g., "main" for Ind Brokerage, "core" for Core Portfolios, "elcie" for Custodial, "checking" for Checking account, "individual" for Brokerage x3379, "stock" for Stock Plan) to update visible data.',
      parameters: {
        type: 'object',
        properties: {
          accountId: {
            type: 'string',
            description: 'The ID of the account to select.'
          }
        },
        required: ['accountId'],
      },
    },
    {
      name: 'setAllocationMode',
      description: 'Switch the allocation display mode on the Portfolios page between "asset" (Asset Class) and "sector" (Sector allocation).',
      parameters: {
        type: 'object',
        properties: {
          mode: {
            type: 'string',
            description: 'The allocation mode: "asset" or "sector".'
          }
        },
        required: ['mode'],
      },
    },
    {
      name: 'searchFinancialInfo',
      description: 'Search for public or private financial information, research, news, and history. Supports specifying a source or querying general sources.',
      parameters: {
        type: 'object',
        properties: {
          query: {
            type: 'string',
            description: 'The search query (e.g. Apple Q3 earnings, portfolio returns, stock valuation)'
          },
          source: {
            type: 'string',
            description: 'Optional preferred source (e.g. yahoo, sec, bloomberg, reuters, morningstar, private_holdings, private_transactions, private_tax)'
          }
        },
        required: ['query'],
      },
    },
  ];
```

**`public/js/gemini.js` lines 353–360 (verbatim):**

```js
  function getFallbackResponse() {
    return '🔑 **AI features require a Gemini API key.**\n\n' +
      'To enable the AI Assistant:\n' +
      '1. Get a free API key at [Google AI Studio](https://aistudio.google.com/apikey)\n' +
      '2. Click the **✨ AI Assistant** button in the top bar\n' +
      '3. Enter your API key and click **Save & Connect**\n\n' +
      'Once connected, I can analyze your portfolio, help with trades, research stocks, and answer questions about your account.';
  }
```

**`public/js/agent-fabric.js` lines 10–23 (verbatim):**

```js
  const sampleQueries = [
    { agent: 'Portfolio Analyst', agentId: 'portfolio-analyst', query: 'Analyze my portfolio allocation' },
    { agent: 'Portfolio Analyst', agentId: 'portfolio-analyst', query: 'Am I too concentrated in tech stocks?' },
    { agent: 'Trade Assistant', agentId: 'trade-assistant', query: 'Help me place a limit order for NVDA' },
    { agent: 'Trade Assistant', agentId: 'trade-assistant', query: 'What order type should I use for AAPL?' },
    { agent: 'Market Research', agentId: 'market-research', query: 'What\'s the current price of NVDA?' },
    { agent: 'Market Research', agentId: 'market-research', query: 'Compare AAPL and MSFT fundamentals' },
    { agent: 'Customer Support', agentId: 'customer-support', query: 'Where are my tax documents?' },
    { agent: 'Customer Support', agentId: 'customer-support', query: 'How do I transfer funds?' },
    { agent: 'Portfolio Analyst', agentId: 'portfolio-analyst', query: 'Suggest rebalancing changes' },
    { agent: 'Trade Assistant', agentId: 'trade-assistant', query: 'Is it a good time to buy TSLA?' },
    { agent: 'Market Research', agentId: 'market-research', query: 'Explain the P/E ratio for GOOGL' },
    { agent: 'Customer Support', agentId: 'customer-support', query: 'What are the trading fees?' },
  ];
```


## Appendix F — Stylesheets (verbatim, authoritative)

Copy these as-is. The server minifies them to `*.min.css` at startup (§3).


### F — styles.css

```css
:root {
  --purple: #3c1384;
  --purple-dark: #25085a;
  --purple-bright: #5631f2;
  --purple-chart: #8e00c8;
  --ink: #2b2b30;
  --muted: #64646b;
  --line: #dedee4;
  --panel: #ffffff;
  --band: #f5f5f5;
  --notice: #eef5ff;
  --positive: #137a27;
  --negative: #d80000;
  --blue: #0669d8;
  --focus: #0b66d8;
  --shadow: 0 18px 45px rgba(25, 20, 40, 0.14);
  --shell: 1200px;
}

* {
  box-sizing: border-box;
}

html {
  color-scheme: light;
  font-family: Arial, Helvetica, sans-serif;
  text-size-adjust: 100%;
  overflow-x: hidden;
}

body {
  margin: 0;
  background: var(--band);
  color: var(--ink);
  font-size: 13px;
  line-height: 1.35;
  overflow-x: hidden;
}

body.menu-open {
  overflow: hidden;
}

a {
  color: var(--purple-bright);
  text-decoration: none;
}

h2 {
  font-size: 18px;
}

a:hover,
button:hover {
  text-decoration: underline;
}

button,
input,
select {
  font: inherit;
}

button {
  cursor: pointer;
}

.skip-link {
  background: var(--purple-dark);
  color: #fff;
  left: 1rem;
  padding: 0.6rem 0.8rem;
  position: absolute;
  top: -3rem;
  z-index: 20;
}

.skip-link:focus {
  top: 1rem;
}

.sr-only {
  height: 1px;
  margin: -1px;
  overflow: hidden;
  position: absolute;
  width: 1px;
  clip: rect(0, 0, 0, 0);
}

.shell {
  margin: 0 auto;
  max-width: var(--shell);
  padding: 0;
  width: 100%;
}

.site-header {
  background: #fff;
  box-shadow: 0 1px 0 rgba(0, 0, 0, 0.1);
  z-index: 10;
}

.market-strip {
  background: #f5f5f5;
  border-bottom: 1px solid #e0e0e0;
  color: #3f3f45;
  font-size: 0.78rem;
}

.market-strip__inner {
  align-items: center;
  display: flex;
  gap: 24px;
  min-height: 21px;
  overflow-x: auto;
  white-space: nowrap;
  font-size: 0.72rem;
  padding: 4px 0;
}

.market-badge {
  display: inline-flex;
  align-items: center;
  font-size: 0.68rem;
  font-weight: 700;
  padding: 2px 6px;
  border-radius: 2px;
  margin-left: 6px;
  line-height: 1;
}

.market-badge--positive {
  background-color: #d1e7dd;
  color: var(--positive);
}

.market-badge--negative {
  background-color: #f8d7da;
  color: var(--negative);
}

.market-strip__timestamp {
  color: #64748b;
  font-size: 0.68rem;
}

.market-strip__actions {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: 12px;
}

.market-strip__admin {
  color: #64748b;
  font-size: 0.72rem;
  font-weight: 500;
  text-decoration: none;
  display: inline-flex;
  align-items: center;
  gap: 4px;
  transition: color 0.15s ease;
}

.market-strip__admin-icon {
  font-size: 0.9rem;
  line-height: 1;
  margin-top: -1px;
}

.market-strip__admin:hover {
  color: #475569;
  text-decoration: underline;
}

.market-strip__admin[aria-current="page"] {
  color: #475569;
  font-weight: 600;
}

.market-strip__separator {
  color: #cbd5e1;
  font-size: 0.72rem;
}

.market-strip__logoff {
  color: var(--purple-bright);
  font-size: 0.72rem;
  font-weight: 600;
  text-decoration: none;
}

.market-strip__logoff:hover {
  text-decoration: underline;
}

.positive {
  color: var(--positive);
}

.negative {
  color: var(--negative);
}

.utility-bar {
  background: #fff;
  padding: 6px 0 0;
}

.utility-bar__inner {
  align-items: center;
  display: flex;
  min-height: 50px;
}

.brand {
  color: #171721;
  display: inline-flex;
  align-items: center;
  line-height: 1;
  min-width: 160px;
  position: relative;
  text-decoration: none;
}

.brand__logo {
  height: 38px;
  width: auto;
  display: block;
}

.brand__sub {
  font-size: 0.68rem;
  font-weight: 400;
  margin-left: 38px;
  margin-top: 3px;
}

.brand__badge {
  border: 1px solid var(--line);
  color: var(--muted);
  font-size: 0.64rem;
  font-weight: 700;
  margin-top: 5px;
  padding: 2px 5px;
  width: max-content;
}

.utility-links {
  align-items: center;
  display: flex;
  gap: 16px;
  margin-left: auto;
}

.etrade-status-badge {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.85rem;
  font-weight: 600;
  color: var(--muted);
  background: rgba(255, 255, 255, 0.05);
  padding: 4px 10px;
  border-radius: 20px;
  border: 1px dashed rgba(255, 255, 255, 0.1);
}

.etrade-status-badge .status-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background-color: #ef4444;
  display: inline-block;
}

.etrade-status-badge .status-dot.online {
  background-color: #22c55e;
  box-shadow: 0 0 8px #22c55e;
}

.etrade-connect-btn {
  background: linear-gradient(135deg, var(--purple-accent), #6d28d9);
  color: #ffffff;
  border: 0;
  font-weight: 700;
  font-size: 0.8rem;
  padding: 6px 14px;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 4px 12px rgba(109, 40, 217, 0.3);
}

.etrade-connect-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 16px rgba(109, 40, 217, 0.4);
}

.etrade-connect-btn.connected {
  background: rgba(255, 255, 255, 0.1);
  color: var(--muted);
  border: 1px solid rgba(255, 255, 255, 0.15);
  box-shadow: none;
}

.utility-links a,
.nav-button {
  align-items: center;
  background: none;
  border: 0;
  color: var(--purple-bright);
  display: inline-flex;
  font-weight: normal;
  min-height: 38px;
  padding: 0;
}

.nav-button::after {
  border-left: 4px solid transparent;
  border-right: 4px solid transparent;
  border-top: 5px solid currentColor;
  content: "";
  display: inline-block;
  margin-left: 6px;
}

.secondary-dropdown::after {
  border-left: 3px solid transparent;
  border-right: 3px solid transparent;
  border-top: 4px solid currentColor;
  content: "";
  display: inline-block;
  margin-left: 4px;
  position: relative;
  top: -1px;
}

.alert-link {
  gap: 6px;
}

.alert-count {
  background: #eb2a00;
  color: #fff;
  display: inline-grid;
  font-size: 0.8rem;
  min-width: 31px;
  padding: 4px 5px;
  place-items: center;
}

.mobile-menu-button {
  background: #fff;
  border: 1px solid var(--line);
  display: none;
  flex-direction: column;
  gap: 4px;
  height: 42px;
  justify-content: center;
  margin-left: auto;
  padding: 0 10px;
  width: 46px;
}

.mobile-menu-button span:not(.sr-only) {
  background: var(--purple-dark);
  display: block;
  height: 2px;
  width: 100%;
}

.primary-nav {
  background: var(--purple-dark);
}

.primary-nav__inner {
  align-items: center;
  display: flex;
  min-height: 38px;
}

.primary-nav a {
  align-items: center;
  color: #fff;
  display: inline-flex;
  font-weight: 500;
  min-height: 38px;
  padding: 0 14px;
  position: relative;
  font-size: 13px;
}

.primary-nav a[aria-current="page"]::after {
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-bottom: 8px solid #fff;
  bottom: 0;
  content: "";
  left: 50%;
  position: absolute;
  transform: translateX(-50%);
}

.search-form {
  margin-left: auto;
  max-width: 180px;
  width: 100%;
  position: relative;
}

.search-form::before {
  content: "";
  position: absolute;
  left: 11px;
  top: 50%;
  width: 10px;
  height: 10px;
  border: 2px solid rgba(255, 255, 255, 0.9);
  border-radius: 50%;
  transform: translateY(-55%);
  z-index: 1;
  pointer-events: none;
}

.search-form::after {
  content: "";
  position: absolute;
  left: 21px;
  top: 55%;
  width: 2px;
  height: 5px;
  background: rgba(255, 255, 255, 0.9);
  transform: rotate(-45deg);
  z-index: 1;
  pointer-events: none;
}

.search-form input {
  background: rgba(255, 255, 255, 0.18);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 15px;
  color: #fff;
  height: 30px;
  padding: 0 12px 0 30px;
  width: 100%;
  font-size: 0.8rem;
}

.search-form input::placeholder {
  color: rgba(255, 255, 255, 0.9);
  font-size: 0.8rem;
}

.secondary-nav {
  background: #fff;
  border-bottom: 1px solid var(--line);
}

.secondary-nav__inner {
  align-items: center;
  display: flex;
  min-height: 32px;
  overflow-x: auto;
  white-space: nowrap;
}

.secondary-nav a,
.secondary-dropdown {
  align-items: center;
  background: none;
  border: 0;
  color: #171721;
  display: inline-flex;
  font-weight: 400;
  font-size: 13px;
  min-height: 32px;
  padding: 0 11px;
  position: relative;
}

.secondary-nav a[aria-current="page"] {
  color: #171721;
  font-weight: 500;
}

.secondary-nav a[aria-current="page"]::after {
  background: var(--purple-bright);
  bottom: 0;
  content: "";
  height: 4px;
  left: 14px;
  position: absolute;
  right: 14px;
}

.menu-popover {
  background: #fff;
  border: 1px solid var(--line);
  box-shadow: var(--shadow);
  min-width: 220px;
  padding: 8px;
  position: fixed;
  right: 160px;
  top: 98px;
  z-index: 30;
}

.menu-popover a {
  color: var(--ink);
  display: block;
  padding: 10px 12px;
}

#app {
  min-height: 650px;
}

#app:focus {
  outline: none;
}

.notice {
  align-items: flex-start;
  background: var(--notice);
  display: flex;
  gap: 12px;
  margin: 20px 0 14px;
  padding: 12px 16px;
}

.notice__icon {
  background: var(--blue);
  border-radius: 50%;
  color: #fff;
  display: inline-grid;
  flex: 0 0 auto;
  font-size: 0.85rem;
  font-weight: 900;
  height: 22px;
  place-items: center;
  width: 22px;
}

.page-band {
  background: #fff;
  padding: 0 0 26px;
}

.page-band--gray {
  background: var(--band);
}

.page-title-row {
  align-items: center;
  display: flex;
  gap: 18px;
  justify-content: space-between;
}

.page-title-row h1,
.welcome h1 {
  font-size: 22px;
  line-height: 1.1;
  margin: 0;
}

.toolbar {
  align-items: center;
  display: flex;
  flex-wrap: wrap;
  gap: 14px;
  justify-content: flex-end;
}

.summary-line {
  align-items: end;
  display: grid;
  gap: 24px;
  grid-template-columns: repeat(4, minmax(0, 1fr)) auto;
  margin-top: 22px;
}

.summary-metric span,
.metric span,
.card-metric span {
  color: var(--muted);
  display: block;
  margin-bottom: 4px;
}

.summary-metric strong,
.metric strong,
.card-metric strong {
  display: block;
  font-size: 22px;
}

.summary-action {
  align-self: start;
  justify-self: end;
}

.button,
.outline-button {
  align-items: center;
  background: #fff;
  border: 1px solid var(--purple-bright);
  color: var(--purple-bright);
  display: inline-flex;
  font-weight: 700;
  justify-content: center;
  min-height: 34px;
  padding: 6px 14px;
  text-align: center;
}

.button {
  background: var(--purple-bright);
  color: #fff;
}

.text-button,
.card-action,
.link-button {
  background: none;
  border: 0;
  color: var(--purple-bright);
  font-weight: normal;
  padding: 0;
}

.icon-button {
  align-items: center;
  background: none;
  border: 0;
  color: var(--purple-bright);
  display: inline-flex;
  font-weight: 900;
  justify-content: center;
  min-height: 34px;
  min-width: 34px;
}

.switch {
  align-items: center;
  display: inline-flex;
  gap: 10px;
  cursor: pointer;
  user-select: none;
}

.switch input {
  position: absolute;
  opacity: 0;
  width: 0;
  height: 0;
}

.switch span {
  background: #b9b9be;
  border-radius: 999px;
  display: inline-flex;
  height: 20px;
  padding: 2px;
  width: 36px;
  position: relative;
  transition: background-color 0.15s ease;
}

.switch span::after {
  background: #fff;
  border-radius: 50%;
  content: "";
  height: 16px;
  transition: transform 0.15s ease;
  width: 16px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.15);
  position: absolute;
  top: 2px;
  left: 2px;
}

.switch input:checked+span {
  background: var(--purple-bright);
}

.switch input:checked+span::after {
  transform: translateX(16px);
}

.dashboard-hero {
  background: #fff;
  padding-bottom: 4px;
}

.dashboard-hero.dashboard-hero--connected {
  /* Keep bottom position static to prevent vertical boundary shifts */
}

.dashboard-controls {
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  gap: 28px;
  align-items: start;
  padding: 20px 0 12px;
}

.welcome p {
  margin: 2px 0 0;
  font-size: 0.65rem;
  color: var(--muted);
}

.metric-row {
  display: flex;
  gap: 50px;
  margin-top: 14px;
}

.welcome .metric-row {
  display: flex;
  align-items: stretch;
  gap: 20px;
  margin-top: 14px;
}

.welcome .metric-row .metric:first-child {
  border-right: 1px solid #dedee4;
  padding-right: 20px;
}

.dashboard-actions {
  align-items: flex-end;
  display: flex;
  flex-direction: column;
  gap: 10px;
  font-size: 0.82rem;
  justify-self: end;
  width: 280px;
}

.dashboard-actions .switch {
  display: inline-flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
}

.dashboard-actions .outline-button {
  width: 100%;
  text-align: center;
  border-radius: 6px;
}

.content-grid {
  display: grid;
  gap: 24px;
  grid-template-columns: minmax(0, 1fr) 340px;
  padding: 24px 0 36px;
}

.content-grid>*,
.account-card__top>*,
.portfolio-summary__grid>*,
.allocation-body>*,
.placeholder-grid>* {
  min-width: 0;
}

.account-stack {
  display: grid;
  gap: 24px;
  min-width: 0;
}

.account-card,
.side-panel,
.wide-panel,
.placeholder-panel {
  background: var(--panel);
  min-width: 0;
  padding: 22px 24px;
}

.account-card__top {
  display: grid;
  gap: 20px;
  grid-template-columns: minmax(0, 1.4fr) minmax(270px, 0.8fr);
}

.account-title {
  align-items: baseline;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  min-width: 0;
  margin: 0 0 28px;
}

.account-title strong {
  color: var(--purple-bright);
  font-size: 16px;
}

.account-title em {
  color: var(--ink);
  font-style: normal;
  font-size: 16px;
  font-weight: 700;
}

.account-title strong,
.account-title em {
  overflow-wrap: anywhere;
}

.beneficiary {
  align-items: flex-start;
  background: var(--notice);
  display: flex;
  gap: 10px;
  margin-top: 14px;
  padding: 14px 16px;
}

.beneficiary .notice__icon {
  height: 20px;
  width: 20px;
}

.account-metrics {
  display: grid;
  gap: 9px;
}

.metric-line {
  display: grid;
  gap: 12px;
  grid-template-columns: minmax(0, 1fr) max-content;
}

.metric-line strong {
  font-size: 16px;
}

.account-card__links {
  align-items: center;
  display: flex;
  flex-wrap: wrap;
  gap: 18px;
  margin-top: 28px;
}

.separator {
  color: var(--muted);
}

.mini-tabs,
.portfolio-tabs,
.alert-tabs {
  align-items: center;
  display: flex;
  flex-wrap: wrap;
  gap: 22px;
  margin-top: 28px;
}

.mini-tabs button,
.portfolio-tabs a,
.alert-tabs button {
  background: none;
  border: 0;
  color: var(--ink);
  font-weight: 700;
  min-height: 36px;
  padding: 0 0 8px;
  position: relative;
}

.mini-tabs button[aria-selected="true"],
.portfolio-tabs a[aria-current="page"],
.alert-tabs button[aria-selected="true"] {
  color: var(--purple-bright);
}

.mini-tabs button[aria-selected="true"]::after,
.portfolio-tabs a[aria-current="page"]::after,
.alert-tabs button[aria-selected="true"]::after {
  background: var(--purple-bright);
  bottom: 0;
  content: "";
  height: 4px;
  left: 0;
  position: absolute;
  right: 0;
}

.table-wrap {
  overflow-x: auto;
  width: 100%;
}

table {
  border-collapse: collapse;
  width: 100%;
}

th,
td {
  border-bottom: 1px solid var(--line);
  padding: 10px 8px;
  text-align: right;
  white-space: nowrap;
}

th:first-child,
td:first-child {
  text-align: left;
}

th {
  border-bottom-width: 3px;
  color: var(--muted);
  font-weight: 500;
}

.table-footer {
  align-items: center;
  display: flex;
  gap: 16px;
  justify-content: space-between;
  margin-top: 16px;
}

.side-rail {
  display: grid;
  gap: 24px;
  min-width: 0;
}

.side-panel h2,
.wide-panel h2 {
  color: var(--purple-bright);
  font-size: 18px;
  margin: 0 0 20px;
}

.side-panel h3 {
  font-size: 16px;
  margin: 0;
}

.side-list {
  display: grid;
  gap: 0;
}

.side-list__item {
  border-bottom: 1px solid var(--line);
  display: grid;
  gap: 6px;
  grid-template-columns: minmax(0, 1fr) auto;
  padding: 15px 0;
}

.side-list__item p {
  margin: 0;
}

.alert-item {
  border-bottom: 1px solid var(--line);
  padding: 14px 0;
}

.alert-item strong {
  color: var(--purple-bright);
  display: block;
}

.alert-item span {
  display: block;
  margin-top: 5px;
}

.footer-disclosures {
  background: var(--band);
  padding: 42px 0 54px;
  text-align: center;
}

.footer-disclosures h2 {
  font-size: 1.2rem;
  margin: 0 0 32px;
}

.disclosure-block {
  border-top: 1px solid var(--line);
  margin: 34px auto 0;
  max-width: 1180px;
  padding-top: 34px;
}

.disclosure-pill {
  border: 1px solid var(--line);
  display: inline-flex;
  flex-wrap: wrap;
  font-weight: 700;
  gap: 20px;
  justify-content: center;
  margin: 16px 0 20px;
  padding: 14px 28px;
}

.disclosure-text {
  margin: 30px auto 0;
  max-width: 1180px;
  text-align: left;
}

.disclosure-links {
  display: flex;
  flex-wrap: wrap;
  gap: 8px 16px;
  margin-top: 24px;
}

.portfolio-header {
  background: #fff;
  padding: 28px 0 0;
}

.portfolio-summary {
  background: var(--band);
  padding: 28px 0;
}

.portfolio-summary__grid {
  display: grid;
  gap: 28px;
  grid-template-columns: 320px repeat(3, minmax(0, 1fr)) auto;
}

.field label {
  color: var(--ink);
  display: block;
  margin-bottom: 8px;
}

.field select {
  background: #fff;
  border: 1px solid #777;
  min-height: 44px;
  padding: 0 12px;
  width: 100%;
}

.allocation-card {
  background: #fff;
  margin: 0 auto;
  max-width: 1500px;
  padding: 30px 40px 52px;
}

.allocation-body {
  align-items: center;
  display: grid;
  gap: 70px;
  grid-template-columns: 380px minmax(0, 620px);
  margin-top: 36px;
}

.donut {
  align-items: center;
  aspect-ratio: 1;
  background: conic-gradient(var(--purple-chart) 0 98.1%, #f58a2b 98.1% 100%);
  border-radius: 50%;
  display: grid;
  justify-items: center;
  max-width: 310px;
  position: relative;
  width: 100%;
}

.donut::after {
  background: #fff;
  border-radius: 50%;
  content: "";
  inset: 28px;
  position: absolute;
}

.donut__label {
  position: relative;
  text-align: center;
  z-index: 1;
}

.donut__label strong {
  display: block;
  font-size: 1.45rem;
}

.legend-line {
  align-items: center;
  display: flex;
  gap: 10px;
}

.legend-line::before {
  background: var(--purple-chart);
  content: "";
  height: 22px;
  width: 4px;
}

.holdings-sections {
  background: #fff;
  margin: 24px auto 0;
  max-width: 1500px;
  padding: 30px 40px 70px;
}

.holding-section+.holding-section {
  margin-top: 58px;
}

.holding-section h3 {
  color: var(--purple-bright);
  font-size: 1.25rem;
  font-weight: 500;
}

.placeholder-panel {
  margin: 30px auto;
  max-width: 1120px;
}

.placeholder-grid {
  display: grid;
  gap: 18px;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  margin-top: 24px;
}

.placeholder-card {
  border: 1px solid var(--line);
  padding: 18px;
}

.chat-launcher {
  align-items: center;
  background: var(--purple-dark);
  border: 0;
  border-radius: 999px;
  bottom: 24px;
  box-shadow: var(--shadow);
  color: #fff;
  display: inline-flex;
  font-weight: 900;
  gap: 9px;
  min-height: 48px;
  padding: 0 22px;
  position: fixed;
  right: 28px;
  z-index: 12;
}

.chat-launcher__icon {
  border: 3px solid #fff;
  display: inline-block;
  height: 18px;
  position: relative;
  width: 24px;
}

.chat-launcher__icon::after {
  border-top: 8px solid #fff;
  border-left: 8px solid transparent;
  bottom: -9px;
  content: "";
  position: absolute;
  right: 2px;
}

.agent-panel {
  background: #fff;
  border: 1px solid var(--line);
  bottom: 84px;
  box-shadow: var(--shadow);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  max-height: min(560px, calc(100vh - 120px));
  position: fixed;
  right: 28px;
  width: min(390px, calc(100vw - 32px));
  z-index: 25;
}

.agent-panel[hidden] {
  display: none;
}

.agent-panel__header {
  align-items: center;
  border-bottom: 1px solid var(--line);
  display: flex;
  justify-content: space-between;
  padding: 16px;
}

.agent-panel__header span {
  color: var(--muted);
  display: block;
  font-size: 0.86rem;
}

.agent-panel__body {
  display: grid;
  gap: 12px;
  overflow: auto;
  padding: 16px;
  flex: 1;
  min-height: 0;
}

.agent-panel__body button {
  background: var(--notice);
  border: 1px solid #c9dfff;
  color: var(--purple-bright);
  font-weight: 700;
  padding: 10px 12px;
  text-align: left;
}

.agent-panel__composer {
  border-top: 1px solid var(--line);
  display: grid;
  gap: 8px;
  grid-template-columns: minmax(0, 1fr) auto;
  padding: 12px;
}

.agent-panel__composer input {
  border: 1px solid var(--line);
  min-height: 40px;
  padding: 0 10px;
}

.agent-panel__composer button {
  background: var(--purple-bright);
  border: 0;
  color: #fff;
  font-weight: 700;
  padding: 0 14px;
}

@media (max-width: 1220px) {

  .content-grid,
  .account-card__top,
  .portfolio-summary__grid,
  .summary-line,
  .dashboard-controls {
    grid-template-columns: 1fr;
  }

  .summary-action,
  .dashboard-actions {
    justify-self: stretch;
  }

  .dashboard-actions {
    width: 100%;
    max-width: 320px;
    align-items: flex-start;
  }

  .dashboard-actions .switch {
    justify-content: space-between;
  }

  .allocation-body {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 920px) {
  .shell {
    padding: 0 16px;
  }

  .site-header {
    position: static;
  }

  .mobile-menu-button {
    display: inline-flex;
  }

  .utility-links {
    display: none;
  }

  .primary-nav__inner {
    align-items: stretch;
    display: none;
    flex-direction: column;
    padding-bottom: 14px;
  }

  .primary-nav.is-open .primary-nav__inner {
    display: flex;
  }

  .primary-nav a {
    justify-content: flex-start;
    padding: 0;
  }

  .primary-nav a[aria-current="page"]::after {
    display: none;
  }

  .search-form {
    max-width: none;
    width: 100%;
  }

  .dashboard-controls,
  .page-title-row,
  .table-footer {
    align-items: flex-start;
    flex-direction: column;
  }

  .metric-row {
    gap: 28px;
  }

  .account-card,
  .side-panel,
  .wide-panel,
  .allocation-card,
  .holdings-sections {
    padding: 22px 16px;
  }

  .placeholder-grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 620px) {
  body {
    font-size: 15px;
  }

  .brand {
    min-width: 0;
  }

  .brand__main {
    font-size: 1.5rem;
  }

  .brand__sub {
    margin-left: 0;
  }

  .market-strip__inner {
    gap: 18px;
  }

  .page-title-row h1,
  .welcome h1 {
    font-size: 1.7rem;
  }

  .metric-row {
    flex-direction: column;
  }

  .account-title strong,
  .account-title em {
    font-size: 1.1rem;
  }

  .metric-line {
    grid-template-columns: 1fr;
  }

  .dashboard-actions {
    align-items: stretch;
  }

  .chat-launcher {
    bottom: 16px;
    right: 16px;
  }

  .agent-panel {
    bottom: 72px;
    right: 16px;
  }
}


/* ═══════════════════════════════
   5. SHARED COMPONENTS
   ═══════════════════════════════ */

/* -- Stat card -- */
.geap-stat-card {
  background: var(--geap-surface);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  transition: all 0.2s ease;
  position: relative;
  overflow: hidden;
}

.geap-stat-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 2px;
  background: linear-gradient(135deg, var(--geap-blue), var(--geap-purple));
  opacity: 0;
  transition: opacity 0.2s ease;
}

.geap-stat-card:hover::before {
  opacity: 1;
}

.geap-stat-card:hover {
  border-color: #cbd5e1;
  transform: translateY(-2px);
  box-shadow: var(--geap-shadow-md);
}

.geap-stat-card .stat-icon {
  width: 40px;
  height: 40px;
  border-radius: var(--geap-radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1rem;
  margin-bottom: 4px;
}

.geap-stat-card .stat-icon.blue {
  background: var(--geap-blue-light);
  color: var(--geap-blue);
}

.geap-stat-card .stat-icon.green {
  background: var(--geap-green-light);
  color: var(--geap-green);
}

.geap-stat-card .stat-icon.purple {
  background: var(--geap-purple-light);
  color: var(--geap-purple);
}

.geap-stat-card .stat-icon.amber {
  background: var(--geap-amber-light);
  color: var(--geap-amber);
}

.geap-stat-card .stat-label {
  font-size: 0.75rem;
  color: var(--geap-text-secondary);
  font-weight: 500;
}

.geap-stat-card .stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: var(--geap-text);
  letter-spacing: -0.02em;
  font-family: 'SF Mono', 'Fira Code', monospace;
}

.geap-stat-card .stat-change {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 0.75rem;
  font-weight: 500;
  font-family: 'SF Mono', 'Fira Code', monospace;
}

.geap-stat-card .stat-change.positive {
  color: var(--geap-green);
}

.geap-stat-card .stat-change.negative {
  color: var(--geap-danger);
}

/* -- Badges -- */
.geap-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  font-size: 0.6875rem;
  font-weight: 600;
  border-radius: var(--geap-radius-pill);
  white-space: nowrap;
  background: var(--geap-surface-tertiary);
  color: var(--geap-text-secondary);
}

.geap-badge--green {
  background: var(--geap-green-light);
  color: #059669;
}

.geap-badge--blue {
  background: var(--geap-blue-light);
  color: var(--geap-blue);
}

.geap-badge--red {
  background: var(--geap-danger-light);
  color: var(--geap-danger);
}

.geap-badge--amber {
  background: var(--geap-amber-light);
  color: #d97706;
}

.geap-badge--purple {
  background: var(--geap-purple-light);
  color: var(--geap-purple);
}

/* -- Badge dot (status indicator) -- */
.geap-badge-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  display: inline-block;
  background: var(--geap-text-tertiary);
}

.geap-badge-dot--active {
  background: var(--geap-green);
  animation: geapPulseGlow 2s infinite;
}

.geap-badge-dot--warning {
  background: var(--geap-amber);
}

.geap-badge-dot--error {
  background: var(--geap-danger);
}

/* -- Flow diagram nodes -- */
.flow-node {
  background: var(--geap-surface);
  border: 2px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  padding: 14px 20px;
  text-align: center;
  min-width: 200px;
  transition: all 0.15s ease;
  cursor: pointer;
  font-size: 0.8125rem;
  color: var(--geap-text);
}

.flow-node:hover {
  border-color: var(--geap-blue);
  box-shadow: 0 0 16px rgba(79, 142, 247, 0.12);
}

.flow-node.active {
  border-color: var(--geap-blue);
  background: var(--geap-blue-light);
}

.flow-node.router {
  border-radius: var(--geap-radius-pill);
  background: linear-gradient(135deg, var(--geap-blue-light), var(--geap-purple-light));
  border-color: var(--geap-purple);
}

.flow-node .node-label {
  font-weight: 600;
  color: var(--geap-text);
}

.flow-node .node-type {
  font-size: 0.6875rem;
  color: var(--geap-text-tertiary);
  margin-top: 2px;
}

.flow-connector {
  width: 2px;
  height: 32px;
  background: var(--geap-border);
  position: relative;
  margin: 0 auto;
}

.flow-connector::after {
  content: '▼';
  position: absolute;
  bottom: -8px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 8px;
  color: var(--geap-text-tertiary);
}

.flow-branches {
  display: flex;
  gap: 16px;
  justify-content: center;
}

/* -- Toggle switch (iOS-style) -- */
.toggle-switch {
  position: relative;
  width: 44px;
  height: 24px;
  background: var(--geap-surface-tertiary);
  border-radius: var(--geap-radius-pill);
  cursor: pointer;
  transition: background 0.15s ease;
  border: 1px solid var(--geap-border);
  flex-shrink: 0;
}

.toggle-switch.active {
  background: var(--geap-blue);
  border-color: var(--geap-blue);
}

.toggle-switch::after {
  content: '';
  position: absolute;
  top: 3px;
  left: 3px;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: #fff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.15);
  transition: transform 0.15s ease;
}

.toggle-switch.active::after {
  transform: translateX(20px);
}

/* -- Tool chip -- */
.tool-chip {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 10px;
  background: var(--geap-surface-secondary);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-pill);
  font-size: 0.6875rem;
  color: var(--geap-text-secondary);
  margin: 2px;
}

.tool-chip .tool-icon {
  font-size: 0.6875rem;
}

/* -- Page header -- */
.geap-page-header {
  padding: 24px 0 20px;
}

.geap-page-header h1 {
  font-size: 1.75rem;
  font-weight: 800;
  color: var(--geap-text);
  margin: 0;
  line-height: 1.2;
}

.geap-page-header .subtitle {
  font-size: 0.875rem;
  color: var(--geap-text-secondary);
  margin-top: 4px;
}

.geap-page-header .actions-row {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 16px;
}

/* -- Content grid -- */
.geap-content-grid {
  display: grid;
  gap: 20px;
}

.geap-content-grid.cols-2 {
  grid-template-columns: repeat(2, 1fr);
}

.geap-content-grid.cols-3 {
  grid-template-columns: repeat(3, 1fr);
}

.geap-content-grid.cols-4 {
  grid-template-columns: repeat(4, 1fr);
}

@media (max-width: 900px) {

  .geap-content-grid.cols-3,
  .geap-content-grid.cols-4 {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 600px) {

  .geap-content-grid.cols-2,
  .geap-content-grid.cols-3,
  .geap-content-grid.cols-4 {
    grid-template-columns: 1fr;
  }
}

/* -- Sparkline container -- */
.sparkline-container {
  width: 80px;
  height: 24px;
  display: inline-block;
  vertical-align: middle;
}

.sparkline-container canvas {
  width: 100%;
  height: 100%;
}


/* ═══════════════════════════════
   6. ANIMATIONS
   ═══════════════════════════════ */

@keyframes geapFadeInUp {
  from {
    opacity: 0;
    transform: translateY(16px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes geapPulseGlow {

  0%,
  100% {
    box-shadow: 0 0 0 0 rgba(52, 211, 153, 0.4);
  }

  50% {
    box-shadow: 0 0 0 6px rgba(52, 211, 153, 0);
  }
}

@keyframes geapTickerScroll {
  0% {
    transform: translateX(0);
  }

  100% {
    transform: translateX(-50%);
  }
}

@keyframes geapTyping {

  0%,
  60%,
  100% {
    opacity: 0.3;
  }

  30% {
    opacity: 1;
  }
}

.animate-fade-in {
  animation: geapFadeInUp 0.3s ease both;
}


/* ═══════════════════════════════
   GEAP INTELLIGENCE STRIP v2
   Premium insight cards with gradients
   ═══════════════════════════════ */

.geap-insights-strip {
  background: transparent;
  border: none;
  padding: 0 0 16px;
}

.geap-insights-strip .insights-header {
  max-width: var(--shell);
  margin: 0 auto;
  padding: 0 0 12px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 8px;
}


.insights-brand {
  display: flex;
  align-items: center;
  gap: 8px;
}

.insights-brand-icon {
  font-size: 14px;
  color: #6B2D9B;
  animation: pulseGlow 2s ease-in-out infinite;
}

@keyframes pulseGlow {

  0%,
  100% {
    opacity: 1;
    transform: scale(1);
  }

  50% {
    opacity: 0.7;
    transform: scale(1.15);
  }
}

.insights-brand-label {
  font-weight: 700;
  font-size: 0.85rem;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  color: #1a1a2e;
}

.insights-live-badge {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 0.7rem;
  font-weight: 600;
  color: #166534;
  background: #dcfce7;
  padding: 2px 8px;
  border-radius: 10px;
}

.insights-live-dot {
  width: 6px;
  height: 6px;
  background: #22c55e;
  border-radius: 50%;
  display: inline-block;
  animation: livePulse 1.5s ease-in-out infinite;
}

@keyframes livePulse {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.4;
  }
}

.insights-subtitle {
  font-size: 0.78rem;
  color: #64748b;
}

.insights-powered {
  font-size: 0.75rem;
  color: #94a3b8;
  display: flex;
  align-items: center;
  gap: 4px;
}

.insights-info-icon {
  font-size: 0.85rem;
  color: #94a3b8;
}

/* ── Main 3-column insight cards ── */

.insights-cards-row {
  max-width: var(--shell);
  margin: 0 auto;
  display: flex;
  align-items: stretch;
  gap: 12px;
  position: relative;
}

.insights-cards-main {
  flex: 1;
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.insights-actions-col {
  display: flex;
  flex-direction: column;
  gap: 8px;
  width: 140px;
  flex-shrink: 0;
}

.insights-view-all-overlay,
.insights-run-diag-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  width: 100%;
  padding: 16px 12px;
  background: #ffffff;
  border: 1px solid #ede9f6;
  border-radius: 16px;
  color: #6B2D9B;
  font-weight: 700;
  font-size: 0.72rem;
  text-align: center;
  text-decoration: none;
  cursor: pointer;
  z-index: 2;
  transition: background 0.2s ease, box-shadow 0.2s ease, color 0.2s ease, border-color 0.2s ease;
  flex: 1;
  box-sizing: border-box;
}

.insights-view-all-overlay:hover,
.insights-run-diag-btn:hover {
  background: #6B2D9B;
  color: #ffffff;
  border-color: #6B2D9B;
  box-shadow: 0 4px 16px rgba(107, 45, 155, 0.12);
  text-decoration: none;
}

.insight-card-v2__icon-new {
  font-size: 1.25rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.insight-card-v2__title-new {
  font-size: 0.95rem;
  font-weight: 700;
  color: #1a1a2e;
}

.insight-card-v2 {
  border-radius: 16px;
  overflow: hidden;
  border: 1px solid #ede9f6;
  background: #ffffff;
  box-shadow: 0 2px 12px rgba(107, 45, 155, 0.06);
  transition: box-shadow 0.25s ease, transform 0.25s ease;
  animation: geapFadeInUp 0.5s cubic-bezier(0.22, 1, 0.36, 1) both;
}

.insight-card-v2:hover {
  box-shadow: 0 8px 28px rgba(107, 45, 155, 0.12);
  transform: translateY(-3px);
}

.insight-card-v2__inner {
  padding: 20px;
  position: relative;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  height: 100%;
}

.insight-card-v2__decor {
  position: absolute;
  right: -10px;
  bottom: -10px;
  width: 160px;
  height: 120px;
  pointer-events: none;
  z-index: 0;
}

.insight-card-v2__header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 12px;
  position: relative;
  z-index: 1;
}

.insight-card-v2__icon {
  width: 38px;
  height: 38px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.7);
  font-size: 18px;
  backdrop-filter: blur(4px);
}

.insight-card-v2__badge {
  font-size: 0.68rem;
  font-weight: 700;
  padding: 3px 10px;
  border-radius: 10px;
  letter-spacing: 0.02em;
  white-space: nowrap;
}

.insight-card-v2__title {
  font-size: 1rem;
  font-weight: 700;
  color: #1a1a2e;
  margin-bottom: 8px;
  position: relative;
  z-index: 1;
}

.insight-card-v2__body {
  display: flex;
  gap: 12px;
  position: relative;
  z-index: 1;
  flex: 1;
}

.insight-card-v2__text {
  font-size: 0.82rem;
  line-height: 1.55;
  color: #374151;
  flex: 1;
}

.insight-card-v2__text strong {
  color: #1a1a2e;
  font-weight: 700;
}

.insight-card-v2__pie {
  flex-shrink: 0;
  display: flex;
  align-items: center;
}

.insight-card-v2__action {
  display: inline-block;
  margin-top: 12px;
  font-size: 0.82rem;
  font-weight: 700;
  color: #6B2D9B;
  text-decoration: none;
  position: relative;
  z-index: 1;
}

.insight-card-v2__action:hover {
  text-decoration: underline;
}

/* ── Bottom row: Mover card + View all ── */

.insights-bottom-row {
  max-width: var(--shell);
  margin: 12px auto 0;
  display: flex;
  gap: 16px;
  align-items: stretch;
}

.insight-mover-card {
  flex: 1;
  display: flex;
  gap: 16px;
  align-items: center;
  background: #fff;
  border: 1px solid #ede9f6;
  border-radius: 16px;
  padding: 16px 20px;
  box-shadow: 0 2px 12px rgba(107, 45, 155, 0.04);
  animation: geapFadeInUp 0.5s cubic-bezier(0.22, 1, 0.36, 1) both;
}

.insight-mover-card__left {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.insight-mover-card__icon {
  width: 42px;
  height: 42px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 12px;
  background: #f5f0ff;
  font-size: 20px;
}

.insight-mover-card__info {
  flex: 1;
  min-width: 0;
}

.insight-mover-card__info .insight-card-v2__title {
  margin-bottom: 4px;
}

.insight-mover-card__info .insight-card-v2__text {
  font-size: 0.8rem;
}

.insight-mover-card__tickers {
  display: flex;
  gap: 12px;
  flex-shrink: 0;
}

.insight-mover-card__separator {
  color: #ede9f6;
  font-size: 24px;
  font-weight: 300;
  margin: 0 16px;
  align-self: center;
  user-select: none;
}

.insight-mover-card__rebalance {
  display: flex;
  gap: 12px;
  align-items: flex-start;
  min-width: 260px;
}

.insight-mover-card__rebalance-icon {
  font-size: 1.4rem;
  flex-shrink: 0;
  margin-top: 2px;
}

.insight-mover-card__rebalance-content h3 {
  font-size: 0.88rem;
  font-weight: 700;
  color: #1a1a2e;
  margin: 0 0 4px;
}

.insight-mover-card__rebalance-content p {
  font-size: 0.78rem;
  color: #475569;
  line-height: 1.35;
  margin: 0 0 4px;
}

.insight-mover-card__rebalance-content a {
  font-size: 0.78rem;
  font-weight: 700;
  color: #6B2D9B;
  text-decoration: none;
}

.insight-mover-card__rebalance-content a:hover {
  text-decoration: underline;
}


.mover-ticker {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 8px 12px;
  background: #faf9ff;
  border: 1px solid #ede9f6;
  border-radius: 10px;
}

.mover-ticker__symbol {
  font-size: 0.78rem;
  font-weight: 700;
  color: #1a1a2e;
}

.mover-ticker__pct {
  font-size: 0.72rem;
  font-weight: 600;
}

.insights-view-all {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 16px 28px;
  background: #fff;
  border: 1px solid #6B2D9B;
  border-radius: 16px;
  color: #6B2D9B;
  font-weight: 700;
  font-size: 0.85rem;
  text-decoration: none;
  white-space: nowrap;
  transition: background 0.2s ease, color 0.2s ease;
  animation: geapFadeInUp 0.5s cubic-bezier(0.22, 1, 0.36, 1) both;
  animation-delay: 400ms;
}

.insights-view-all:hover {
  background: #6B2D9B;
  color: #fff;
  text-decoration: none;
}

.insights-view-all__icon {
  font-size: 1rem;
}

@keyframes geapFadeInUp {
  from {
    opacity: 0;
    transform: translateY(12px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@media (max-width: 920px) {
  .insights-cards-main {
    grid-template-columns: 1fr;
    padding: 0 16px;
  }

  .insights-bottom-row {
    flex-direction: column;
    padding: 0 16px;
  }

  .insight-mover-card {
    flex-direction: column;
    align-items: flex-start;
    gap: 12px;
  }

  .insight-mover-card__tickers {
    width: 100%;
    justify-content: flex-start;
  }

  .insight-mover-card__separator {
    display: none;
  }

  .insight-mover-card__rebalance {
    min-width: 0;
    width: 100%;
    border-top: 1px solid #ede9f6;
    padding-top: 12px;
  }

  .geap-insights-strip .insights-header {
    padding: 0 16px 16px;
  }
}

/* ═══════════════════════════════
   MARKETS TODAY BOX
   ═══════════════════════════════ */

.markets-today-box {
  display: flex;
  align-items: center;
  gap: 16px;
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 16px 20px;
  max-width: 340px;
  justify-self: center;
}

.markets-today-box__content h3 {
  font-size: 0.9rem;
  font-weight: 700;
  color: #1a1a2e;
  margin: 0 0 6px;
}

.markets-today-box__content p {
  font-size: 0.78rem;
  color: #475569;
  line-height: 1.4;
  margin: 0 0 8px;
}

.markets-today-box__chart {
  flex-shrink: 0;
}

.metric__value-row {
  display: flex;
  align-items: center;
  gap: 12px;
}

.metric__value-text {
  display: flex;
  align-items: baseline;
  gap: 6px;
  white-space: nowrap;
}

.metric-sparkline {
  flex-shrink: 0;
}

.welcome .metric strong {
  display: inline-block;
}

.metric__pct {
  display: inline-block;
  font-size: 0.82rem;
  font-weight: 700;
  margin-left: 6px;
}

.metric__pct.positive {
  color: var(--positive) !important;
}

.metric__pct.negative {
  color: var(--negative) !important;
}

/* ═══════════════════════════════
   PORTFOLIO SUMMARY GRID
   ═══════════════════════════════ */

.portfolio-summary-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 20px;
  padding: 24px 0;
}

.portfolio-summary-card {
  background: #fff;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  padding: 20px;
}

.portfolio-summary-card h3 {
  font-size: 1rem;
  font-weight: 700;
  color: #1a1a2e;
  margin: 0 0 4px;
}

.portfolio-summary-card__sub {
  font-size: 0.78rem;
  color: #64748b;
  margin: 0 0 16px;
}

.portfolio-summary-card__link {
  display: inline-block;
  margin-top: 16px;
  font-size: 0.82rem;
  font-weight: 700;
  color: #6B2D9B;
  text-decoration: none;
}

.portfolio-summary-card__link:hover {
  text-decoration: underline;
}

/* Holdings Overview */
.holdings-overview {
  display: flex;
  align-items: center;
  gap: 16px;
}

.holdings-donut {
  flex-shrink: 0;
}

.holdings-list {
  flex: 1;
}

.holdings-list__header {
  font-size: 0.72rem;
  font-weight: 600;
  color: #94a3b8;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  margin-bottom: 8px;
}

.holdings-list__row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 4px 0;
  font-size: 0.82rem;
}

.holdings-list__icon {
  width: 18px;
  text-align: center;
  font-size: 0.7rem;
  color: #6B2D9B;
}

.holdings-list__name {
  flex: 1;
  color: #1a1a2e;
}

.holdings-list__pct {
  font-weight: 700;
  color: #1a1a2e;
}

/* Sector Bars */
.sector-bars {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.sector-bar-row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.sector-bar-row__name {
  font-size: 0.8rem;
  color: #374151;
  min-width: 120px;
}

.sector-bar-row__track {
  flex: 1;
  height: 8px;
  background: #f1f5f9;
  border-radius: 4px;
  overflow: hidden;
}

.sector-bar-row__fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.6s cubic-bezier(0.22, 1, 0.36, 1);
}

.sector-bar-row__pct {
  font-size: 0.8rem;
  font-weight: 700;
  color: #1a1a2e;
  min-width: 36px;
  text-align: right;
}

/* Recent Activity */
.recent-activity-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.recent-activity-row {
  display: flex;
  align-items: center;
  gap: 12px;
  padding-bottom: 10px;
  border-bottom: 1px solid #f1f5f9;
}

.recent-activity-row:last-child {
  border-bottom: 0;
  padding-bottom: 0;
}

.recent-activity-row__desc {
  flex: 1;
  font-size: 0.82rem;
  color: #1a1a2e;
  font-weight: 500;
}

.recent-activity-row__date {
  font-size: 0.75rem;
  color: #94a3b8;
}

.recent-activity-row__amt {
  font-size: 0.82rem;
  font-weight: 700;
  text-align: right;
  min-width: 80px;
}

@media (max-width: 920px) {
  .portfolio-summary-grid {
    grid-template-columns: 1fr;
  }

  .markets-today-box {
    max-width: 100%;
  }
}

/* ═══════════════════════════════
   METRIC LINKS
   ═══════════════════════════════ */

.metric__link {
  display: block;
  font-size: 0.78rem;
  font-weight: 600;
  color: #6B2D9B;
  text-decoration: none;
  margin-top: 6px;
}

.metric__link:hover {
  text-decoration: underline;
}

/* ═══════════════════════════════
   DASHBOARD ACTIONS ROW
   ═══════════════════════════════ */

.dashboard-actions__row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.dashboard-actions__row a {
  font-size: 0.82rem;
  color: var(--focus);
  text-decoration: none;
  font-weight: 700;
}

.dashboard-actions__row a:hover {
  text-decoration: underline;
}

.dashboard-actions__row .separator {
  color: #d1d5db;
}

.dashboard-actions__row .text-button {
  font-size: 0.82rem;
  color: var(--focus);
  font-weight: 700;
}

/* ═══════════════════════════════
   AI INSIGHT + ACCOUNT GRID
   ═══════════════════════════════ */

.ai-account-section {
  background: #f5f3ff;
  border-top: 1px solid #ede9f6;
  border-bottom: 1px solid #ede9f6;
}

.ai-account-grid {
  display: grid;
  grid-template-columns: 1fr 300px;
  gap: 20px;
  padding: 14px 0;
  align-items: start;
}

.ai-account-grid__left {
  min-width: 0;
}

/* ── Compact Account Summary ── */

.compact-account {
  border-left: 1px solid #ede9f6;
  padding-left: 20px;
}

.compact-account__header {
  font-size: 0.85rem;
  margin-bottom: 12px;
}

.compact-account__header strong {
  color: #1a1a2e;
}

.compact-account__header em {
  color: #64748b;
  font-style: normal;
  margin: 0 4px;
}

.compact-account__header .link-button {
  font-size: 0.75rem;
  color: #6B2D9B;
}

.compact-account__metrics {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 10px;
}

.compact-account__row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.8rem;
}

.compact-account__row span {
  color: #64748b;
}

.compact-account__row strong {
  color: #1a1a2e;
  font-weight: 600;
}

.compact-account__notice {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin-top: 12px;
  padding: 10px;
  background: #f0f4ff;
  border-radius: 8px;
  font-size: 0.75rem;
  color: #374151;
  line-height: 1.4;
}

.compact-account__notice .notice-icon {
  color: #3b82f6;
  font-size: 0.9rem;
  flex-shrink: 0;
}

.compact-account__notice .link-button {
  font-size: 0.75rem;
  color: #6B2D9B;
}

/* ═══════════════════════════════
   AI INSIGHT STRIP
   ═══════════════════════════════ */

.ai-insight-strip {
  background: transparent;
}

.ai-insight-strip__inner {
  display: flex;
  align-items: center;
  gap: 20px;
  padding: 12px 0;
}

.ai-insight-strip__left {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
  min-width: 0;
}

.ai-insight-strip__icon {
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #ffffff;
  color: #6B2D9B;
  border: 1px solid #ede9f6;
  border-radius: 50%;
  font-size: 18px;
  flex-shrink: 0;
  box-shadow: 0 2px 6px rgba(107, 45, 155, 0.08);
}

.ai-insight-strip__content {
  min-width: 0;
}

.ai-insight-strip__label {
  font-weight: 700;
  font-size: 0.85rem;
  color: #1a1a2e;
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 2px;
}

.ai-insight-strip__text {
  font-size: 0.8rem;
  color: #374151;
  line-height: 1.4;
}

.ai-insight-strip__text strong {
  color: #1a1a2e;
}

.ai-insight-strip__actions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}

.ai-action-pill {
  display: flex;
  flex-direction: row;
  align-items: center;
  text-align: left;
  padding: 10px 14px;
  background: #fff;
  border: 1px solid #ede9f6;
  border-radius: 10px;
  text-decoration: none;
  color: #1a1a2e;
  font-size: 0.75rem;
  font-weight: 600;
  line-height: 1.3;
  transition: box-shadow 0.2s ease, border-color 0.2s ease;
  gap: 12px;
}

.ai-action-pill:hover {
  border-color: #6B2D9B;
  box-shadow: 0 2px 8px rgba(107, 45, 155, 0.1);
  text-decoration: none;
}

.ai-action-pill__icon {
  font-size: 1.25rem;
  margin-bottom: 0;
  flex-shrink: 0;
}

.ai-action-pill__sub {
  font-size: 0.7rem;
  font-weight: 600;
  color: #6B2D9B;
}

.ai-insight-strip__close {
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: none;
  border: 1px solid #d1d5db;
  border-radius: 50%;
  color: #64748b;
  cursor: pointer;
  font-size: 0.75rem;
  flex-shrink: 0;
  transition: background 0.15s;
}

.ai-insight-strip__close:hover {
  background: #f1f5f9;
}

/* ═══════════════════════════════
   TEAM SUMMARY CARD
   ═══════════════════════════════ */

.team-summary {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

.team-summary__info p {
  margin: 0 0 2px;
  font-size: 0.8rem;
  color: #64748b;
}

.team-summary__info p strong {
  color: #1a1a2e;
  font-size: 0.85rem;
}

.team-summary__phone {
  font-size: 0.82rem;
  font-weight: 600;
  color: #6B2D9B;
  text-decoration: none;
}

.team-summary__phone:hover {
  text-decoration: underline;
}

.team-summary__avatar {
  width: 44px;
  height: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f5f0ff;
  border-radius: 50%;
  font-size: 1.4rem;
  flex-shrink: 0;
}

@media (max-width: 920px) {
  .ai-insight-strip__inner {
    flex-direction: column;
    align-items: flex-start;
  }

  .ai-insight-strip__actions {
    flex-wrap: wrap;
  }
}

/* ── Markets & AI Insights Card ── */
.markets-ai-card {
  display: flex;
  flex-direction: column;
  align-self: stretch;
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 14px 18px;
  max-width: 540px;
  width: 100%;
  justify-self: center;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  transition: background-color 0.08s cubic-bezier(0.4, 0, 0.2, 1) 0.3s,
    border-color 0.08s cubic-bezier(0.4, 0, 0.2, 1) 0.3s,
    margin-bottom 0.1s cubic-bezier(0.4, 0, 0.2, 1) 0.2s,
    padding-bottom 0.1s cubic-bezier(0.4, 0, 0.2, 1) 0.2s,
    border-bottom-left-radius 0.08s cubic-bezier(0.4, 0, 0.2, 1) 0.3s,
    border-bottom-right-radius 0.08s cubic-bezier(0.4, 0, 0.2, 1) 0.3s;
}

.markets-ai-card.markets-ai-card--connected {
  background: #ffffff;
  border-bottom-left-radius: 12px;
  border-bottom-right-radius: 12px;
  position: relative;
  z-index: 10;
  margin-bottom: 0;
  padding-bottom: 14px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  border-color: #e2e8f0;
  border-bottom-color: #e2e8f0;
  transition: background-color 0.08s cubic-bezier(0.4, 0, 0.2, 1) 0s,
    border-color 0.08s cubic-bezier(0.4, 0, 0.2, 1) 0s,
    margin-bottom 0.1s cubic-bezier(0.4, 0, 0.2, 1) 0.08s,
    padding-bottom 0.1s cubic-bezier(0.4, 0, 0.2, 1) 0.08s,
    border-bottom-left-radius 0.08s cubic-bezier(0.4, 0, 0.2, 1) 0s,
    border-bottom-right-radius 0.08s cubic-bezier(0.4, 0, 0.2, 1) 0s;
}

.markets-ai-card--connected .markets-ai-card__header {
  border-bottom-color: #e2e8f0;
}

.markets-ai-card--connected .markets-ai-card__section--ai {
  border-left-color: #cbd5e1;
}

.markets-ai-card--connected .markets-ai-card__toggle-btn {
  background: #ffffff;
  border-color: #e2e8f0;
}

.markets-ai-card--connected .markets-ai-card__toggle-btn:hover {
  background: #f8fafc;
}

/* Concave rounded corner joints (inwards) using radial gradients */
.markets-ai-card::before,
.markets-ai-card::after {
  content: "";
  position: absolute;
  bottom: -1px;
  width: 17px;
  height: 17px;
  pointer-events: none;
  z-index: 11;
  opacity: 0;
  transition: opacity 0.1s cubic-bezier(0.4, 0, 0.2, 1) 0.2s;
}

.markets-ai-card::before {
  left: -16px;
  background: radial-gradient(circle at 0 0, transparent 16px, #eef7fc 16.5px);
}

.markets-ai-card::after {
  right: -16px;
  background: radial-gradient(circle at 100% 0, transparent 16px, #eef7fc 16.5px);
}

.markets-ai-card.markets-ai-card--connected::before,
.markets-ai-card.markets-ai-card--connected::after {
  opacity: 0;
  transition: opacity 0.1s cubic-bezier(0.4, 0, 0.2, 1) 0.08s;
}

.markets-ai-card__header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 4px;
}

.markets-ai-card__brand {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.85rem;
  color: #1a1a2e;
}

.markets-ai-card__star {
  color: #6B2D9B;
  font-size: 1.1rem;
}

.markets-ai-card__toggle-btn {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  border: 1px solid #e2e8f0;
  background: #ffffff;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #64748b;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
  padding: 0;
}

.markets-ai-card__toggle-btn:hover {
  background: #f8fafc;
  color: #1e293b;
}

.markets-ai-card__content {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
}

.markets-ai-card__content .markets-ai-card__section:last-child {
  margin-top: auto;
}

.markets-ai-card__section {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 0.78rem;
  color: #475569;
  line-height: 1.4;
}

.markets-ai-card__market-desc {
  margin: 0;
  font-size: 0.78rem;
  color: #475569;
  line-height: 1.4;
}

.markets-ai-card--connected .markets-ai-card__market-desc {
  color: #334155;
}

.markets-ai-card__divider {
  border-top: 1px solid #e2e8f0;
  margin: 4px 0;
}

.markets-ai-card--connected .markets-ai-card__divider {
  border-top-color: #cbd5e1;
}

.markets-ai-card__ai-desc {
  margin: 0 0 6px;
  color: #374151;
}

.markets-ai-card__ai-desc strong {
  color: #1a1a2e;
}

.ai-details-section {
  background: #ffffff;
  border-bottom: 1px solid #ede9f6;
  padding: 12px 0;
  display: grid;
  grid-template-rows: 1fr;
  overflow: hidden;
  transition: grid-template-rows 0.2s cubic-bezier(0.4, 0, 0.2, 1) 0.18s,
    padding 0.2s cubic-bezier(0.4, 0, 0.2, 1) 0.18s,
    border-color 0.2s cubic-bezier(0.4, 0, 0.2, 1) 0.18s,
    border-bottom-width 0.2s cubic-bezier(0.4, 0, 0.2, 1) 0.18s;
}

.ai-details-section--collapsed {
  grid-template-rows: 0fr;
  padding-top: 0;
  padding-bottom: 0;
  border-bottom-color: transparent;
  border-bottom-width: 0;
  transition: grid-template-rows 0.2s cubic-bezier(0.4, 0, 0.2, 1) 0s,
    padding 0.2s cubic-bezier(0.4, 0, 0.2, 1) 0s,
    border-color 0.2s cubic-bezier(0.4, 0, 0.2, 1) 0s,
    border-bottom-width 0.2s cubic-bezier(0.4, 0, 0.2, 1) 0s;
}

.ai-details-inner {
  min-height: 0;
  overflow: hidden;
  transition: opacity 0.15s ease 0.22s;
  opacity: 1;
}

.ai-details-section--collapsed .ai-details-inner {
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.08s ease 0s;
}

.ai-details-card {
  background: #f8f7ff;
  border: 1px solid #ede9f6;
  border-radius: 16px;
  padding: 20px 24px;
  position: relative;
  box-shadow: 0 2px 8px rgba(107, 45, 155, 0.03);
}

.ai-details-card__header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 18px;
}

.ai-details-card__title {
  font-size: 1.1rem;
  font-weight: 700;
  color: #1a1a2e;
  margin: 0 0 4px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.ai-details-card__subtitle {
  font-size: 0.8rem;
  color: #64748b;
  margin: 0;
}

.ai-details-card__close-btn {
  background: none;
  border: 0;
  color: #94a3b8;
  font-size: 1.1rem;
  cursor: pointer;
  padding: 4px;
  transition: color 0.15s;
}

.ai-details-card__close-btn:hover {
  color: #475569;
}

.ai-details-card__grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 24px;
}

.ai-details-card__col {
  display: flex;
  gap: 12px;
  align-items: flex-start;
}

.ai-details-card__col:not(:last-child) {
  border-right: 1px solid #ede9f6;
  padding-right: 20px;
}

.ai-details-card__icon {
  font-size: 1.4rem;
  flex-shrink: 0;
}

.ai-details-card__col-content h3 {
  font-size: 0.88rem;
  font-weight: 700;
  color: #1a1a2e;
  margin: 0 0 6px;
}

.ai-details-card__col-content p {
  font-size: 0.78rem;
  color: #475569;
  line-height: 1.45;
  margin: 0 0 8px;
}

.ai-details-card__col-content a {
  font-size: 0.78rem;
  font-weight: 700;
  color: #6B2D9B;
  text-decoration: none;
}

.ai-details-card__col-content a:hover {
  text-decoration: underline;
}

@media (max-width: 920px) {
  .ai-details-card__grid {
    grid-template-columns: 1fr;
    gap: 16px;
  }

  .ai-details-card__col:not(:last-child) {
    border-right: none;
    border-bottom: 1px solid #ede9f6;
    padding-right: 0;
    padding-bottom: 16px;
  }
}


/* ═══════════════════════════════════════════════════════════════════════════
   RESPONSIVE GRIDS FOR PLANNING & RISK ASSESSMENT
   ═══════════════════════════════════════════════════════════════════════════ */

.planning-stats-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
  margin-bottom: 24px;
}

.planning-tools-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
}

.risk-factor-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
  margin-bottom: 24px;
}

@media (max-width: 1024px) {
  .planning-stats-grid {
    grid-template-columns: 1fr 1fr;
  }
}

@media (max-width: 768px) {
  .planning-tools-grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 480px) {

  .planning-stats-grid,
  .risk-factor-grid {
    grid-template-columns: 1fr;
  }
}

/* Skeletons */
.geap-skeleton-text {
  height: 12px;
  background: #e2e8f0;
  border-radius: 4px;
  animation: geapPulse 1.5s infinite ease-in-out;
  display: inline-block;
}

.geap-skeleton-title {
  height: 24px;
  background: #e2e8f0;
  border-radius: 4px;
  animation: geapPulse 1.5s infinite ease-in-out;
  display: inline-block;
}

.geap-skeleton-line {
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  animation: geapPulse 1.5s infinite ease-in-out;
  display: block;
}

.geap-skeleton-circle {
  border-radius: 50%;
  background: #e2e8f0;
  animation: geapPulse 1.5s infinite ease-in-out;
  display: inline-block;
}

.geap-skeleton-card {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  animation: geapPulse 1.5s infinite ease-in-out;
}

@keyframes geapPulse {

  0%,
  100% {
    opacity: 0.6;
  }

  50% {
    opacity: 0.35;
  }
}```

### F — chat.css

```css
/* ═══════════════════════════════════════════════════════════════════════════
   GEAP AGENT COMPONENT STYLES
   Ported from GEAP-POC (dark theme) → GEAP light theme
   ═══════════════════════════════════════════════════════════════════════════ */


/* ═══════════════════════════════
   1. GEAP CSS CUSTOM PROPERTIES
   ═══════════════════════════════ */

:root {
  /* ── Agent accent colors ── */
  --geap-blue: #4F8EF7;
  --geap-green: #34D399;
  --geap-purple: #A78BFA;
  --geap-amber: #FBBF24;
  --geap-danger: #EF4444;

  /* ── Surface colors (light theme) ── */
  --geap-surface: #ffffff;
  --geap-surface-secondary: #f8f9fa;
  --geap-surface-tertiary: #f1f3f5;
  --geap-border: #e2e8f0;
  --geap-text: #1a202c;
  --geap-text-secondary: #64748b;
  --geap-text-tertiary: #94a3b8;

  /* ── Derived tokens ── */
  --geap-blue-light: rgba(79, 142, 247, 0.10);
  --geap-green-light: rgba(52, 211, 153, 0.10);
  --geap-purple-light: rgba(167, 139, 250, 0.10);
  --geap-amber-light: rgba(251, 191, 36, 0.10);
  --geap-danger-light: rgba(239, 68, 68, 0.10);
  --geap-shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.06);
  --geap-shadow-md: 0 4px 12px rgba(0, 0, 0, 0.08);
  --geap-shadow-lg: 0 8px 24px rgba(0, 0, 0, 0.10);
  --geap-radius-sm: 6px;
  --geap-radius-md: 8px;
  --geap-radius-lg: 12px;
  --geap-radius-pill: 9999px;
  --geap-chat-width: 380px;
}


/* ═══════════════════════════════
   2. CHAT / AGENT PANEL STYLES
   ═══════════════════════════════ */

/* -- Override existing .agent-panel to full-featured sliding chat -- */
.agent-panel.geap-chat {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  width: var(--geap-chat-width);
  max-height: none;
  height: 100vh;
  background: var(--geap-surface);
  border: none;
  border-left: 1px solid var(--geap-border);
  box-shadow: -4px 0 24px rgba(0, 0, 0, 0.08);
  display: flex;
  flex-direction: column;
  z-index: 30;
  transform: translateX(100%);
  transition: transform 0.35s cubic-bezier(0.22, 1, 0.36, 1);
}

.agent-panel.geap-chat.open {
  transform: translateX(0);
}

.agent-panel.geap-chat[hidden] {
  display: none;
}

/* -- Chat header -- */
.agent-panel.geap-chat .agent-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 16px;
  border-bottom: 1px solid var(--geap-border);
  background: var(--geap-surface);
  flex-shrink: 0;
}

/* -- Agent selector -- */
.agent-selector {
  display: flex;
  gap: 6px;
  padding: 8px 16px;
  border-bottom: 1px solid var(--geap-border);
  overflow-x: auto;
  flex-shrink: 0;
  background: var(--geap-surface-secondary);
}

.agent-selector .agent-chip {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 12px;
  font-size: 0.75rem;
  font-weight: 500;
  color: var(--geap-text-secondary);
  background: var(--geap-surface);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-pill);
  cursor: pointer;
  white-space: nowrap;
  transition: all 0.15s ease;
}

.agent-selector .agent-chip:hover {
  background: var(--geap-surface-tertiary);
  color: var(--geap-text);
}

.agent-selector .agent-chip.active {
  background: var(--geap-blue-light);
  color: var(--geap-blue);
  border-color: rgba(79, 142, 247, 0.3);
}

/* -- Chat messages area -- */
.chat-messages {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-height: 0;
}

/* -- Individual chat message -- */
.chat-message {
  display: flex;
  gap: 10px;
  animation: geapFadeInUp 0.25s cubic-bezier(0.22, 1, 0.36, 1);
}

.chat-message.user {
  flex-direction: row-reverse;
}

/* -- Message avatar -- */
.message-avatar {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.7rem;
  font-weight: 700;
  flex-shrink: 0;
}

.chat-message.assistant .message-avatar {
  background: linear-gradient(135deg, var(--geap-blue), var(--geap-purple));
  color: #fff;
}

.chat-message.user .message-avatar {
  background: var(--geap-surface-tertiary);
  color: var(--geap-text);
}

/* -- Message content bubble -- */
.message-content {
  max-width: 85%;
  padding: 10px 14px;
  border-radius: var(--geap-radius-lg);
  font-size: 0.8125rem;
  line-height: 1.55;
}

.chat-message.assistant .message-content {
  background: var(--geap-surface-secondary);
  color: var(--geap-text);
  border: 1px solid var(--geap-border);
  border-bottom-left-radius: var(--geap-radius-sm);
}

.chat-message.user .message-content {
  background: var(--geap-blue);
  color: #fff;
  border-bottom-right-radius: var(--geap-radius-sm);
}

.message-content p+p {
  margin-top: 8px;
}

.message-content strong {
  font-weight: 600;
}

.message-content code {
  font-family: 'SF Mono', 'Fira Code', monospace;
  font-size: 0.7rem;
  background: rgba(0, 0, 0, 0.06);
  padding: 1px 4px;
  border-radius: 3px;
}

.chat-message.user .message-content code {
  background: rgba(255, 255, 255, 0.2);
}

/* -- Streaming / typing indicator -- */
.chat-streaming {
  display: flex;
  gap: 4px;
  padding: 10px 14px;
}

.chat-streaming .dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--geap-text-tertiary);
  animation: geapTyping 1.4s infinite;
}

.chat-streaming .dot:nth-child(2) {
  animation-delay: 0.2s;
}

.chat-streaming .dot:nth-child(3) {
  animation-delay: 0.4s;
}

/* -- Chat input area -- */
.chat-input-area {
  padding: 12px 16px;
  border-top: 1px solid var(--geap-border);
  flex-shrink: 0;
  background: var(--geap-surface);
}

.chat-input-area .input-wrapper {
  display: flex;
  align-items: flex-end;
  gap: 8px;
  background: var(--geap-surface-secondary);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  padding: 8px 12px;
  transition: border-color 0.15s ease;
}

.chat-input-area .input-wrapper:focus-within {
  border-color: var(--geap-blue);
  box-shadow: 0 0 0 3px var(--geap-blue-light);
}

.chat-input-area textarea {
  flex: 1;
  border: none;
  background: transparent;
  color: var(--geap-text);
  font-family: inherit;
  font-size: 0.8125rem;
  padding: 4px 0;
  resize: none;
  outline: none;
  max-height: 120px;
  line-height: 1.5;
}

.chat-input-area textarea::placeholder {
  color: var(--geap-text-tertiary);
}

.chat-input-area .send-btn {
  width: 32px;
  height: 32px;
  border-radius: var(--geap-radius-md);
  background: linear-gradient(135deg, var(--geap-blue), var(--geap-purple));
  border: none;
  color: #fff;
  font-size: 0.875rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: transform 0.15s ease, box-shadow 0.15s ease;
}

.chat-input-area .send-btn:hover {
  transform: scale(1.05);
  box-shadow: 0 0 12px rgba(79, 142, 247, 0.3);
}

.chat-input-area .send-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
  transform: none;
}

/* -- Suggestion chips -- */
.suggestion-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  padding: 6px 0;
}

.suggestion-chip {
  padding: 4px 12px;
  font-size: 0.7rem;
  color: var(--geap-blue);
  background: var(--geap-blue-light);
  border: 1px solid rgba(79, 142, 247, 0.2);
  border-radius: var(--geap-radius-pill);
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.suggestion-chip:hover {
  background: rgba(79, 142, 247, 0.18);
  border-color: rgba(79, 142, 247, 0.4);
}



/* ═══════════════════════════════
   OVERRIDES & FIXES
   Make GEAP styles work without .geap-chat qualifier
   ═══════════════════════════════ */

/* Chat typing indicator */
.chat-typing {
  display: flex;
  gap: 4px;
  padding: 4px 0;
}

.chat-typing .dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--geap-text-tertiary);
  animation: geapTyping 1.4s infinite;
}

.chat-typing .dot:nth-child(2) {
  animation-delay: 0.2s;
}

.chat-typing .dot:nth-child(3) {
  animation-delay: 0.4s;
}

/* Chat input within agent-panel__composer */
.agent-panel__composer textarea.chat-input {
  border: 1px solid var(--geap-border, #e2e8f0);
  border-radius: var(--geap-radius-sm, 6px);
  min-height: 36px;
  padding: 8px 10px;
  font-size: 13px;
  font-family: inherit;
  resize: none;
  outline: none;
}

.agent-panel__composer textarea.chat-input:focus {
  border-color: var(--geap-blue, #4F8EF7);
  box-shadow: 0 0 0 2px var(--geap-blue-light, rgba(79, 142, 247, 0.1));
}

.chat-send-btn {
  background: var(--purple-bright, #6B2D9B);
  border: 0;
  color: #fff;
  font-weight: 700;
  padding: 0 14px;
  border-radius: var(--geap-radius-sm, 6px);
  cursor: pointer;
  font-size: 13px;
}

.chat-send-btn:hover {
  opacity: 0.9;
}

/* Chat agent avatar in header */
.chat-agent-avatar {
  font-size: 20px;
}

/* Modal overlay (API key dialog) */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
  animation: geapFadeInUp 0.2s ease;
}

.modal-dialog {
  background: #fff;
  border-radius: var(--geap-radius-lg, 12px);
  padding: 28px;
  max-width: 440px;
  width: 90%;
  box-shadow: var(--geap-shadow-lg, 0 8px 24px rgba(0, 0, 0, 0.1));
}

.modal-dialog h2 {
  font-size: 1.25rem;
  margin: 0 0 12px;
  color: var(--ink, #1a1a2e);
}

.modal-dialog p {
  font-size: 0.875rem;
  color: var(--geap-text-secondary, #64748b);
  margin: 0 0 8px;
}

.modal-dialog a {
  color: var(--purple-bright, #6B2D9B);
}

/* GEAP nav link styling */
.geap-nav-link {
  position: relative;
  margin-left: 2px;
  color: aqua !important;
}

.geap-nav-link--admin {
  color: #ff79c6 !important;
}


/* ═══════════════════════════════════════════════════════════════════════════
   CHAT WIDGETS & REBALANCE INTERACTION
   ═══════════════════════════════════════════════════════════════════════════ */

.chat-rebalance-card {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-top: 4px solid #6B2D9B;
  border-radius: 12px;
  padding: 16px;
  margin: 12px 0;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  font-family: inherit;
}

.chat-rebalance-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 14px;
  font-size: 0.85rem;
}

.rebalance-trade-row {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 12px;
  padding: 8px;
  background: #f8fafc;
  border-radius: 8px;
  border: 1px solid #f1f5f9;
}

.trade-side-badge {
  font-size: 0.65rem;
  font-weight: 800;
  padding: 3px 6px;
  border-radius: 4px;
  min-width: 42px;
  text-align: center;
  letter-spacing: 0.05em;
}

.trade-side-badge.sell {
  background: #fee2e2;
  color: #ef4444;
  border: 1px solid #fecaca;
}

.trade-side-badge.buy {
  background: #dcfce7;
  color: #22c55e;
  border: 1px solid #bbf7d0;
}

.trade-details {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.symbol-line {
  font-size: 0.85rem;
  display: flex;
  align-items: center;
  gap: 6px;
}

.holding-name {
  font-size: 0.7rem;
  color: #64748b;
}

.trade-inputs {
  font-size: 0.75rem;
  color: #475569;
  display: flex;
  align-items: center;
  gap: 6px;
}

.rebalance-input {
  width: 54px;
  padding: 2px 4px;
  border: 1px solid #cbd5e1;
  border-radius: 4px;
  font-size: 0.75rem;
  text-align: center;
  background: #fff;
  color: #0f172a;
  outline: none;
}

.rebalance-input:focus {
  border-color: #6B2D9B;
  box-shadow: 0 0 0 2px rgba(107, 45, 155, 0.1);
}

.price-multiplier {
  color: #94a3b8;
}

.trade-value {
  font-size: 0.85rem;
  font-weight: 700;
  color: #0f172a;
}

.rebalance-divider {
  border-bottom: 1px dashed #e2e8f0;
  margin: 12px 0;
}

.rebalance-summary {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 14px;
  font-size: 0.8rem;
}

.summary-line {
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #475569;
}

.net-positive {
  color: #16a34a;
  font-weight: 700;
}

.net-negative {
  color: #dc2626;
  font-weight: 700;
}

.rebalance-execute-btn {
  width: 100%;
  padding: 10px;
  background: #6B2D9B;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  font-size: 0.8rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  justify-content: center;
  align-items: center;
  box-shadow: 0 2px 4px rgba(107, 45, 155, 0.2);
}

.rebalance-execute-btn:hover {
  background: #56227c;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(107, 45, 155, 0.3);
}

.rebalance-execute-btn:active {
  transform: translateY(0);
}

/* Spinner anim */
.rebalance-spinner {
  width: 16px;
  height: 16px;
  border: 2px solid #cbd5e1;
  border-top-color: #6B2D9B;
  border-radius: 50%;
  animation: rebalance-spin 0.8s linear infinite;
}

@keyframes rebalance-spin {
  to {
    transform: rotate(360deg);
  }
}

.rebalance-status-container {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 12px;
  font-size: 0.75rem;
  color: #475569;
}

/* Success state styles */
.rebalance-success-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 12px 8px;
  animation: rebalance-fade-in 0.35s ease-out;
}

.success-icon-wrapper {
  width: 48px;
  height: 48px;
  margin-bottom: 12px;
}

.rebalance-success-state h3 {
  font-size: 0.95rem;
  font-weight: 700;
  color: #1e293b;
  margin: 0 0 6px 0;
}

.success-desc {
  font-size: 0.75rem;
  color: #64748b;
  line-height: 1.4;
  margin: 0 0 14px 0;
}

.success-details {
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 8px;
  padding: 10px 14px;
  width: 100%;
  font-size: 0.7rem;
  color: #166534;
  text-align: left;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

@keyframes rebalance-fade-in {
  from {
    opacity: 0;
    transform: translateY(8px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Checkmark animation */
.success-checkmark {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  display: block;
  stroke-width: 2;
  stroke: #22c55e;
  stroke-miterlimit: 10;
  box-shadow: inset 0px 0px 0px #22c55e;
  animation: checkmark-fill .4s ease-in-out .4s forwards, checkmark-scale .3s ease-in-out 0s both;
}

.success-checkmark__circle {
  stroke-dasharray: 166;
  stroke-dashoffset: 166;
  stroke-width: 2;
  stroke-miterlimit: 10;
  stroke: #22c55e;
  fill: none;
  animation: checkmark-stroke .6s cubic-bezier(0.65, 0, 0.45, 1) forwards;
}

.success-checkmark__check {
  transform-origin: 50% 50%;
  stroke-dasharray: 48;
  stroke-dashoffset: 48;
  stroke: #22c55e;
  animation: checkmark-stroke .3s cubic-bezier(0.65, 0, 0.45, 1) .8s forwards;
}

@keyframes checkmark-stroke {
  100% {
    stroke-dashoffset: 0;
  }
}

@keyframes checkmark-scale {

  0%,
  100% {
    transform: none;
  }

  50% {
    transform: scale3d(1.1, 1.1, 1);
  }
}

@keyframes checkmark-fill {
  100% {
    box-shadow: inset 0px 0px 0px 30px rgba(34, 197, 94, 0.05);
  }
}

/* Retirement Chart Tab Buttons */
.retirement-tab-btn {
  background: transparent;
  border: none;
  border-bottom: 2px solid transparent;
  color: #64748b;
  font-size: 0.75rem;
  padding: 4px 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  outline: none;
  font-weight: 500;
  font-family: inherit;
}

.retirement-tab-btn:hover {
  color: #1e293b;
  border-bottom-color: #cbd5e1;
}

.retirement-tab-btn.active {
  color: #6B2D9B;
  border-bottom-color: #6B2D9B;
  font-weight: 700;
}

/* Speed Dial Quick Action Buttons */
.speed-dial-btn {
  background: #fdf2f8;
  color: #db2777;
  border: 1px solid #fbcfe8;
  padding: 4px 10px;
  border-radius: 9999px;
  font-size: 0.7rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
  display: flex;
  align-items: center;
}

.speed-dial-btn:hover {
  background: #fbcfe8;
  transform: translateY(-1px);
}

.speed-dial-btn:active {
  transform: translateY(0);
}

/* ═══════════════════════════════
   VOICE & AVATAR CAPABILITIES
   ═══════════════════════════════ */

/* Header controls container */
.agent-panel__header-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

/* Audio Toggle Button in Header */
.chat-audio-toggle {
  background: transparent;
  border: none;
  font-size: 1.1rem;
  cursor: pointer;
  padding: 4px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s ease, transform 0.15s ease;
  color: var(--geap-text-secondary);
}
.chat-audio-toggle:hover {
  background: var(--geap-surface-tertiary);
  transform: scale(1.05);
  text-decoration: none;
}
.chat-audio-toggle.active {
  color: #6B2D9B;
}

/* Composer adjustments to accommodate microphone button */
.agent-panel__composer {
  position: relative;
  display: flex;
  gap: 6px;
  align-items: stretch;
}
.chat-voice-btn {
  background: transparent;
  border: 1px solid var(--geap-border, #e2e8f0);
  border-radius: var(--geap-radius-sm, 6px);
  width: 38px;
  height: 38px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.1rem;
  color: var(--geap-text-secondary);
  cursor: pointer;
  transition: all 0.2s ease;
}
.chat-voice-btn:hover {
  background: var(--geap-surface-tertiary);
  border-color: #cbd5e1;
  text-decoration: none;
}
.chat-voice-btn.listening {
  background: #fee2e2;
  border-color: #fca5a5;
  color: #ef4444;
  animation: voicePulse 1.5s infinite ease-in-out;
}

@keyframes voicePulse {
  0% { box-shadow: 0 0 0 0 rgba(239, 68, 68, 0.4); }
  70% { box-shadow: 0 0 0 6px rgba(239, 68, 68, 0); }
  100% { box-shadow: 0 0 0 0 rgba(239, 68, 68, 0); }
}

/* Neural Core Avatar Container */
.neural-avatar {
  position: relative;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--geap-surface-secondary, #f8f9fa);
  border: 1px solid var(--geap-border, #e2e8f0);
  flex-shrink: 0;
  box-shadow: var(--geap-shadow-sm);
  overflow: hidden;
}

.neural-core {
  position: absolute;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: linear-gradient(135deg, var(--geap-blue, #4F8EF7), var(--geap-purple, #A78BFA));
  transition: all 0.3s ease;
}

/* Dynamic states for core */
.neural-avatar.idle .neural-core {
  animation: coreBreathe 3s infinite ease-in-out;
}

.neural-avatar.thinking .neural-core {
  width: 18px;
  height: 18px;
  background: linear-gradient(225deg, #ef4444, #FBBF24, #34D399, #4F8EF7, #A78BFA);
  background-size: 300% 300%;
  animation: coreSpin 1.5s infinite linear;
}

.neural-avatar.speaking .neural-core {
  width: 8px;
  height: 8px;
  opacity: 0.5;
  animation: coreBreathe 2s infinite ease-in-out;
}

/* EQ visual sound wave bars */
.neural-waves {
  display: none;
  position: absolute;
  bottom: 6px;
  left: 50%;
  transform: translateX(-50%);
  align-items: flex-end;
  gap: 2px;
  height: 18px;
  z-index: 1;
}

.neural-avatar.speaking .neural-waves {
  display: flex;
}

.wave-bar {
  width: 2px;
  background: linear-gradient(to top, var(--geap-blue), var(--geap-purple));
  border-radius: 1px;
  height: 2px;
}

.neural-avatar.speaking .wave-bar.bar-1 { animation: wavePulseHeight 0.6s infinite ease-in-out alternate; }
.neural-avatar.speaking .wave-bar.bar-2 { animation: wavePulseHeight 0.9s infinite ease-in-out alternate 0.1s; }
.neural-avatar.speaking .wave-bar.bar-3 { animation: wavePulseHeight 0.5s infinite ease-in-out alternate 0.2s; }
.neural-avatar.speaking .wave-bar.bar-4 { animation: wavePulseHeight 0.8s infinite ease-in-out alternate 0.15s; }
.neural-avatar.speaking .wave-bar.bar-5 { animation: wavePulseHeight 0.7s infinite ease-in-out alternate 0.05s; }

@keyframes coreBreathe {
  0%, 100% { transform: scale(0.9); box-shadow: 0 0 4px rgba(79,142,247,0.2); }
  50% { transform: scale(1.1); box-shadow: 0 0 10px rgba(167,139,250,0.6); }
}

@keyframes coreSpin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

@keyframes wavePulseHeight {
  from { height: 3px; }
  to { height: 16px; }
}

```

### F — insights.css

```css
/* ═══════════════════════════════════════════════════════════════
   GEAP — AI Insights Hub Page Styles
   ═══════════════════════════════════════════════════════════════ */

/* Agent Team Grid */
.agent-team-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
  margin-bottom: 24px;
}

.agent-card {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05), 0 2px 4px -1px rgba(0, 0, 0, 0.02);
  display: flex;
  flex-direction: column;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.agent-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.08), 0 4px 6px -2px rgba(0, 0, 0, 0.04);
}

.agent-card__header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.agent-card__icon {
  font-size: 1.5rem;
}

.agent-card__status {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.7rem;
  font-weight: 700;
  color: #16a34a;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  background: #f0fdf4;
  padding: 3px 8px;
  border-radius: 9999px;
  border: 1px solid #dcfce7;
}

/* Status Pulse Animation */
.active-dot-pulse {
  width: 6px;
  height: 6px;
  background-color: #22c55e;
  border-radius: 50%;
  position: relative;
  display: inline-block;
}

.active-dot-pulse::after {
  content: '';
  width: 100%;
  height: 100%;
  background-color: #22c55e;
  border-radius: 50%;
  position: absolute;
  top: 0;
  left: 0;
  animation: status-pulse 1.6s ease-in-out infinite;
}

@keyframes status-pulse {
  0% {
    transform: scale(1);
    opacity: 1;
  }

  100% {
    transform: scale(2.8);
    opacity: 0;
  }
}

.agent-card__name {
  font-size: 0.95rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 6px 0;
}

.agent-card__desc {
  font-size: 0.75rem;
  line-height: 1.45;
  color: #475569;
  margin: 0 0 16px 0;
  flex: 1;
}

.agent-card__footer {
  font-size: 0.65rem;
  color: #94a3b8;
  border-top: 1px solid #f1f5f9;
  padding-top: 10px;
  font-weight: 500;
}

/* Insights Dashboard Grid */
.ai-insights-dashboard-grid {
  display: grid;
  grid-template-columns: 1.2fr 1fr;
  gap: 28px;
  align-items: start;
}

/* Feed Cards */
.insight-feed-card {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 16px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05);
}

.insight-feed-card__header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.insight-badge {
  font-size: 0.65rem;
  font-weight: 700;
  padding: 3px 8px;
  border-radius: 6px;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  border: 1px solid transparent;
}

.insight-badge.risk {
  background: #fee2e2;
  color: #ef4444;
  border-color: #fecaca;
}

.insight-badge.progress {
  background: #dcfce7;
  color: #16a34a;
  border-color: #bbf7d0;
}

.insight-badge.tax {
  background: #eff6ff;
  color: #2563eb;
  border-color: #dbeafe;
}

.insight-feed-btn {
  background: #fdf2f8;
  color: #db2777;
  border: 1px solid #fbcfe8;
  padding: 8px 16px;
  border-radius: 8px;
  font-size: 0.75rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s ease;
  width: auto;
  display: inline-flex;
  align-items: center;
  font-family: inherit;
}

.insight-feed-btn:hover {
  background: #fbcfe8;
  transform: translateY(-1px);
}

.insight-feed-btn:active {
  transform: translateY(0);
}

/* Sandbox Styling */
.sandbox-card {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 24px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05);
}

.sandbox-form {
  display: grid;
  grid-template-columns: 1fr 1.2fr 1fr;
  gap: 12px;
  margin-bottom: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.form-group label {
  font-size: 0.7rem;
  font-weight: 700;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.form-select,
.form-input {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  font-size: 0.8rem;
  font-weight: 600;
  color: #0f172a;
  background: #ffffff;
  outline: none;
  font-family: inherit;
  transition: border-color 0.15s ease;
}

.form-select:focus,
.form-input:focus {
  border-color: #6B2D9B;
  box-shadow: 0 0 0 2px rgba(107, 45, 155, 0.1);
}

.sandbox-action-btn {
  width: 100%;
  padding: 10px;
  background: #6B2D9B;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  font-size: 0.8rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: inherit;
  box-shadow: 0 2px 4px rgba(107, 45, 155, 0.15);
}

.sandbox-action-btn:hover {
  background: #56227c;
  transform: translateY(-1px);
}

.sandbox-action-btn:active {
  transform: translateY(0);
}

/* Stepper Animation */
.sandbox-stepper {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.stepper-item {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 0.75rem;
  font-weight: 500;
  transition: all 0.2s ease;
}

.stepper-item.pending {
  color: #94a3b8;
}

.stepper-item.completed {
  color: #1e293b;
  font-weight: 600;
}

.stepper-item .step-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #cbd5e1;
  display: inline-block;
  transition: all 0.2s ease;
}

.stepper-item.completed .step-dot {
  background: #22c55e;
  box-shadow: 0 0 0 3px rgba(34, 197, 148, 0.2);
}

/* Results metrics */
.sandbox-results-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-bottom: 12px;
}

.result-metric {
  background: #ffffff;
  border: 1px solid #dcfce7;
  border-radius: 8px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.metric-label {
  font-size: 0.65rem;
  font-weight: 600;
  color: #166534;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.metric-value {
  font-size: 0.95rem;
  font-weight: 800;
  color: #14532d;
}

.sandbox-chat-cta {
  width: 100%;
  padding: 8px 12px;
  background: #22c55e;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  font-size: 0.75rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: inherit;
}

.sandbox-chat-cta:hover {
  background: #16a34a;
  transform: translateY(-1px);
}

/* Responsive Overrides */
@media (max-width: 1024px) {
  .agent-team-grid {
    grid-template-columns: repeat(2, 1fr);
  }

  .ai-insights-dashboard-grid {
    grid-template-columns: 1fr;
    gap: 24px;
  }
}

@media (max-width: 600px) {
  .agent-team-grid {
    grid-template-columns: 1fr;
  }

  .sandbox-form {
    grid-template-columns: 1fr;
  }
}```

### F — settings.css

```css
/* ═══════════════════════════════════════════════════════════════
   GEAP — Customer AI Settings & Guardrails Styles
   ═══════════════════════════════════════════════════════════════ */

.settings-grid {
  display: grid;
  grid-template-columns: 1.6fr 1fr;
  gap: 24px;
}

@media (max-width: 900px) {
  .settings-grid {
    grid-template-columns: 1fr;
  }
}

.settings-group {
  margin-bottom: 24px;
  border-bottom: 1px solid #f1f5f9;
  padding-bottom: 20px;
}

.settings-group:last-child {
  border-bottom: none;
  padding-bottom: 0;
}

.settings-group-title {
  font-size: 15px;
  font-weight: 700;
  color: #1e293b;
  margin: 0 0 12px 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

.settings-options-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

/* Custom Radio Card Selector */
.radio-card {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 14px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease-in-out;
}

.radio-card:hover {
  background: #f1f5f9;
  border-color: #cbd5e1;
}

.radio-card input[type="radio"] {
  margin-top: 3px;
  accent-color: #6B2D9B;
  width: 16px;
  height: 16px;
  flex-shrink: 0;
}

.radio-card-content {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.radio-card-title {
  font-size: 13px;
  font-weight: 700;
  color: #1e293b;
}

.radio-card-desc {
  font-size: 11px;
  color: #64748b;
  line-height: 1.4;
}

/* Active State for selected Radio Cards */
.radio-card.active {
  background: #fbf7ff;
  border-color: #d8b4fe;
  box-shadow: 0 0 0 1px #d8b4fe;
}

/* Custom Switch Toggle */
.switch-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 14px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  transition: all 0.2s ease;
}

.switch-row:hover {
  background: #f1f5f9;
}

.switch-row-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.switch-row-title {
  font-size: 13px;
  font-weight: 700;
  color: #1e293b;
}

.switch-row-desc {
  font-size: 11px;
  color: #64748b;
  line-height: 1.3;
}

/* Information Source Checkbox List */
.source-checkbox-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px;
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.source-checkbox-item:hover {
  background: #f8fafc;
  border-color: #cbd5e1;
}

.source-checkbox-label {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 13px;
  font-weight: 600;
  color: #334155;
  cursor: pointer;
}

.source-checkbox-logo-icon {
  font-size: 16px;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f1f5f9;
  border-radius: 4px;
}

.source-checkbox-badge {
  font-size: 10px;
  font-weight: 700;
  padding: 2px 6px;
  border-radius: 4px;
}

.source-checkbox-badge.public {
  background: #ecfdf5;
  color: #059669;
  border: 1px solid #a7f3d0;
}

.source-checkbox-badge.private {
  background: #eff6ff;
  color: #2563eb;
  border: 1px solid #bfdbfe;
}

/* Save Indicator Toast */
.settings-save-toast {
  position: fixed;
  bottom: 24px;
  right: 24px;
  background: #1e293b;
  color: #fff;
  padding: 12px 20px;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 600;
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.3);
  display: flex;
  align-items: center;
  gap: 8px;
  z-index: 1000;
  opacity: 0;
  transform: translateY(20px);
  transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  pointer-events: none;
}

.settings-save-toast.show {
  opacity: 1;
  transform: translateY(0);
}

.toast-checkmark {
  color: #22c55e;
  font-size: 16px;
}
```

### F — agent-studio.css

```css
/* ═══════════════════════════════
   3. AGENT STUDIO STYLES
   ═══════════════════════════════ */

.studio-layout {
  display: grid;
  grid-template-columns: 240px 1fr 280px;
  gap: 20px;
  height: calc(100vh - 200px);
  min-height: 500px;
}

/* -- Left sidebar: agent list -- */
.studio-sidebar {
  background: var(--geap-surface);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

.studio-sidebar-title {
  font-size: 0.75rem;
  font-weight: 600;
  color: var(--geap-text-secondary);
  padding: 16px 16px 8px;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.studio-agent-list {
  padding: 8px;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.studio-agent-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  border-radius: var(--geap-radius-md);
  cursor: pointer;
  transition: all 0.15s ease;
  border: 1px solid transparent;
}

.studio-agent-item:hover {
  background: var(--geap-surface-secondary);
}

.studio-agent-item.active {
  background: var(--geap-blue-light);
  border-color: rgba(79, 142, 247, 0.2);
}

.studio-agent-icon {
  width: 36px;
  height: 36px;
  border-radius: var(--geap-radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.875rem;
  flex-shrink: 0;
}

.studio-agent-name {
  font-size: 0.8125rem;
  font-weight: 600;
  color: var(--geap-text);
}

.studio-agent-desc {
  font-size: 0.6875rem;
  color: var(--geap-text-tertiary);
}

/* -- Center canvas -- */
.studio-canvas {
  background: var(--geap-surface);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  padding: 20px;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
}

.studio-canvas-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
  padding-bottom: 16px;
  border-bottom: 1px solid var(--geap-border);
}

.studio-canvas-header h2 {
  font-size: 1.1rem;
  font-weight: 700;
  color: var(--geap-text);
  margin: 0;
}

/* -- Right config panel -- */
.studio-config {
  background: var(--geap-surface);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  display: flex;
  flex-direction: column;
  overflow-y: auto;
}

.studio-config-section {
  padding: 16px;
  border-bottom: 1px solid var(--geap-border);
}

.studio-config-section:last-child {
  border-bottom: none;
}

.studio-config-title {
  font-size: 0.6875rem;
  font-weight: 600;
  color: var(--geap-text-tertiary);
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 12px;
}

/* -- Test console -- */
.studio-test-console {
  background: var(--geap-surface-secondary);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-md);
  margin-top: 16px;
  overflow: hidden;
}

.test-console-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 16px;
  background: var(--geap-surface);
  border-bottom: 1px solid var(--geap-border);
  font-size: 0.8125rem;
  font-weight: 600;
  color: var(--geap-text);
}

.test-console-body {
  padding: 16px;
  max-height: 200px;
  overflow-y: auto;
  font-size: 0.8125rem;
  color: var(--geap-text-secondary);
  font-family: 'SF Mono', 'Fira Code', monospace;
}

/* -- Studio responsive -- */
@media (max-width: 1100px) {
  .studio-layout {
    grid-template-columns: 1fr;
    height: auto;
  }

  .studio-sidebar,
  .studio-config {
    max-height: 300px;
  }
}

```

### F — agent-fabric.css

```css
/* ═══════════════════════════════
   4. AGENT FABRIC STYLES
   ═══════════════════════════════ */

.fabric-metrics-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
  margin-bottom: 24px;
}

.fabric-agents-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
}

.fabric-agent-card {
  background: var(--geap-surface);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  padding: 20px;
  transition: all 0.2s ease;
  position: relative;
  overflow: hidden;
}

.fabric-agent-card:hover {
  border-color: #cbd5e1;
  transform: translateY(-2px);
  box-shadow: var(--geap-shadow-md);
}

.fabric-agent-header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
}

.fabric-agent-header .agent-icon {
  width: 40px;
  height: 40px;
  border-radius: var(--geap-radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.1rem;
  flex-shrink: 0;
}

.fabric-agent-header .agent-name {
  font-size: 0.9rem;
  font-weight: 600;
  color: var(--geap-text);
}

.fabric-agent-header .agent-status {
  font-size: 0.7rem;
  color: var(--geap-text-tertiary);
}

.fabric-agent-metrics {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.fabric-metric {
  text-align: center;
}

.fabric-metric .metric-value {
  font-size: 1rem;
  font-weight: 700;
  font-family: 'SF Mono', 'Fira Code', monospace;
  color: var(--geap-text);
}

.fabric-metric .metric-label {
  font-size: 0.6875rem;
  color: var(--geap-text-tertiary);
  margin-top: 2px;
}

/* -- Activity log -- */
.fabric-activity-log {
  background: var(--geap-surface);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  overflow: hidden;
}

.activity-log-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 20px;
  border-bottom: 1px solid var(--geap-border);
  font-weight: 600;
  color: var(--geap-text);
}

.activity-log-body {
  max-height: 400px;
  overflow-y: auto;
}

.log-entry {
  display: grid;
  grid-template-columns: 110px 110px 1fr 80px;
  gap: 12px;
  padding: 10px 20px;
  border-bottom: 1px solid var(--geap-surface-tertiary);
  font-size: 0.8125rem;
  align-items: center;
  transition: background 0.12s ease;
}

.log-entry:hover {
  background: var(--geap-surface-secondary);
}

.log-entry:last-child {
  border-bottom: none;
}

.log-timestamp {
  font-family: 'SF Mono', 'Fira Code', monospace;
  font-size: 0.6875rem;
  color: var(--geap-text-tertiary);
}

.log-agent {
  font-weight: 500;
  color: var(--geap-text);
}

.log-query {
  color: var(--geap-text-secondary);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* -- Governance panel -- */
.governance-panel {
  background: var(--geap-surface);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  padding: 20px;
}

.governance-rule {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 0;
  border-bottom: 1px solid var(--geap-surface-tertiary);
}

.governance-rule:last-child {
  border-bottom: none;
}

.governance-rule .rule-label {
  font-size: 0.8125rem;
  color: var(--geap-text);
}

.governance-rule .rule-desc {
  font-size: 0.75rem;
  color: var(--geap-text-tertiary);
}

/* -- Routing map -- */
.routing-map {
  background: var(--geap-surface);
  border: 1px solid var(--geap-border);
  border-radius: var(--geap-radius-lg);
  padding: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 300px;
}

.routing-map-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

/* -- Fabric responsive -- */
@media (max-width: 1100px) {
  .fabric-agents-grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 768px) {
  .fabric-metrics-grid {
    grid-template-columns: repeat(2, 1fr);
  }

  .log-entry {
    grid-template-columns: 1fr;
    gap: 4px;
  }
}

```


## Appendix G — index.html shell (verbatim; logo data URI elided)

Replace `LOGO_DATA_URI` with your own logo asset.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="description" content="Internal GEAP proof-of-concept mock E*TRADE workspace.">
    <title>GEAP E*TRADE POC</title>
    <script>
      window.loadedScripts = new Set();
      window.ensureScript = function(src, callback) {
        const cleanSrc = src.split('?')[0];
        if (window.loadedScripts.has(cleanSrc) || document.querySelector(`script[src^="${cleanSrc}"]`)) {
          if (callback) callback();
          return;
        }
        const script = document.createElement("script");
        script.src = src;
        script.onload = () => {
          window.loadedScripts.add(cleanSrc);
          if (callback) callback();
        };
        script.onerror = () => {
          if (callback) callback();
        };
        document.body.appendChild(script);
      };

      window.loadedStyles = new Set();
      window.ensureStyle = function(href, callback) {
        const cleanHref = href.split('?')[0];
        if (window.loadedStyles.has(cleanHref) || document.querySelector(`link[href^="${cleanHref}"]`)) {
          if (callback) callback();
          return;
        }
        const link = document.createElement("link");
        link.rel = "stylesheet";
        link.href = href;
        link.onload = () => {
          window.loadedStyles.add(cleanHref);
          if (callback) callback();
        };
        link.onerror = () => {
          if (callback) callback();
        };
        document.head.appendChild(link);
      };
    </script>
    <style>
:root {
  --purple: #3c1384;
  --purple-dark: #25085a;
  --purple-bright: #5631f2;
  --purple-chart: #8e00c8;
  --ink: #2b2b30;
  --muted: #64646b;
  --line: #dedee4;
  --panel: #ffffff;
  --band: #f5f5f5;
  --notice: #eef5ff;
  --positive: #137a27;
  --negative: #d80000;
  --blue: #0669d8;
  --focus: #0b66d8;
  --shadow: 0 18px 45px rgba(25, 20, 40, 0.14);
  --shell: 1200px;
}

* {
  box-sizing: border-box;
}

html {
  color-scheme: light;
  font-family: Arial, Helvetica, sans-serif;
  text-size-adjust: 100%;
  overflow-x: hidden;
}

body {
  margin: 0;
  background: var(--band);
  color: var(--ink);
  font-size: 14px;
  line-height: 1.35;
  overflow-x: hidden;
}

body.menu-open {
  overflow: hidden;
}

a {
  color: var(--purple-bright);
  text-decoration: none;
}

a:hover,
button:hover {
  text-decoration: underline;
}

button,
input,
select {
  font: inherit;
}

button {
  cursor: pointer;
}

.skip-link {
  background: var(--purple-dark);
  color: #fff;
  left: 1rem;
  padding: 0.6rem 0.8rem;
  position: absolute;
  top: -3rem;
  z-index: 20;
}

.skip-link:focus {
  top: 1rem;
}

.sr-only {
  height: 1px;
  margin: -1px;
  overflow: hidden;
  position: absolute;
  width: 1px;
  clip: rect(0, 0, 0, 0);
}

.shell {
  margin: 0 auto;
  max-width: var(--shell);
  padding: 0;
  width: 100%;
}

.site-header {
  background: #fff;
  box-shadow: 0 1px 0 rgba(0, 0, 0, 0.1);
  z-index: 10;
}

.market-strip {
  background: #f5f5f5;
  border-bottom: 1px solid #e0e0e0;
  color: #3f3f45;
  font-size: 0.78rem;
}

.market-strip__inner {
  align-items: center;
  display: flex;
  gap: 24px;
  min-height: 21px;
  overflow-x: auto;
  white-space: nowrap;
  font-size: 0.72rem;
  padding: 4px 0;
}

.market-badge {
  display: inline-flex;
  align-items: center;
  font-size: 0.68rem;
  font-weight: 700;
  padding: 2px 6px;
  border-radius: 2px;
  margin-left: 6px;
  line-height: 1;
}

.market-badge--positive {
  background-color: #d1e7dd;
  color: var(--positive);
}

.market-badge--negative {
  background-color: #f8d7da;
  color: var(--negative);
}

.market-strip__timestamp {
  color: #64748b;
  font-size: 0.68rem;
}

.market-strip__actions {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: 12px;
}

.market-strip__admin {
  color: #64748b;
  font-size: 0.72rem;
  font-weight: 500;
  text-decoration: none;
  display: inline-flex;
  align-items: center;
  gap: 4px;
  transition: color 0.15s ease;
}

.market-strip__admin-icon {
  font-size: 0.9rem;
  line-height: 1;
  margin-top: -1px;
}

.market-strip__admin:hover {
  color: #475569;
  text-decoration: underline;
}

.market-strip__admin[aria-current="page"] {
  color: #475569;
  font-weight: 600;
}

.market-strip__separator {
  color: #cbd5e1;
  font-size: 0.72rem;
}

.market-strip__logoff {
  color: var(--purple-bright);
  font-size: 0.72rem;
  font-weight: 600;
  text-decoration: none;
}

.market-strip__logoff:hover {
  text-decoration: underline;
}

.positive {
  color: var(--positive);
}

.negative {
  color: var(--negative);
}

.utility-bar {
  background: #fff;
}

.utility-bar__inner {
  align-items: center;
  display: flex;
  min-height: 50px;
}

.brand {
  color: #171721;
  display: inline-flex;
  align-items: center;
  line-height: 1;
  min-width: 160px;
  position: relative;
  text-decoration: none;
}

.brand__logo {
  height: 38px;
  width: auto;
  display: block;
}

.brand__sub {
  font-size: 0.68rem;
  font-weight: 400;
  margin-left: 38px;
  margin-top: 3px;
}

.brand__badge {
  border: 1px solid var(--line);
  color: var(--muted);
  font-size: 0.64rem;
  font-weight: 700;
  margin-top: 5px;
  padding: 2px 5px;
  width: max-content;
}

.utility-links {
  align-items: center;
  display: flex;
  gap: 22px;
  margin-left: auto;
}

.utility-links a,
.nav-button {
  align-items: center;
  background: none;
  border: 0;
  color: var(--purple-bright);
  display: inline-flex;
  font-weight: 700;
  min-height: 38px;
  padding: 0;
}

.nav-button::after {
  border-left: 4px solid transparent;
  border-right: 4px solid transparent;
  border-top: 5px solid currentColor;
  content: "";
  display: inline-block;
  margin-left: 6px;
}

.secondary-dropdown::after {
  border-left: 3px solid transparent;
  border-right: 3px solid transparent;
  border-top: 4px solid currentColor;
  content: "";
  display: inline-block;
  margin-left: 4px;
  position: relative;
  top: -1px;
}

.alert-link {
  gap: 6px;
}

.alert-count {
  background: #eb2a00;
  color: #fff;
  display: inline-grid;
  font-size: 0.8rem;
  min-width: 31px;
  padding: 4px 5px;
  place-items: center;
}

.mobile-menu-button {
  background: #fff;
  border: 1px solid var(--line);
  display: none;
  flex-direction: column;
  gap: 4px;
  height: 42px;
  justify-content: center;
  margin-left: auto;
  padding: 0 10px;
  width: 46px;
}

.mobile-menu-button span:not(.sr-only) {
  background: var(--purple-dark);
  display: block;
  height: 2px;
  width: 100%;
}

.primary-nav {
  background: var(--purple-dark);
}

.primary-nav__inner {
  align-items: center;
  display: flex;
  min-height: 38px;
}

.primary-nav a {
  align-items: center;
  color: #fff;
  display: inline-flex;
  font-weight: 500;
  min-height: 38px;
  padding: 0 14px;
  position: relative;
  font-size: 0.85rem;
}

.primary-nav a[aria-current="page"]::after {
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-bottom: 8px solid #fff;
  bottom: 0;
  content: "";
  left: 50%;
  position: absolute;
  transform: translateX(-50%);
}

.search-form {
  margin-left: auto;
  max-width: 180px;
  width: 100%;
  position: relative;
}

.search-form::before {
  content: "";
  position: absolute;
  left: 11px;
  top: 50%;
  width: 10px;
  height: 10px;
  border: 2px solid rgba(255, 255, 255, 0.9);
  border-radius: 50%;
  transform: translateY(-55%);
  z-index: 1;
  pointer-events: none;
}

.search-form::after {
  content: "";
  position: absolute;
  left: 21px;
  top: 55%;
  width: 2px;
  height: 5px;
  background: rgba(255, 255, 255, 0.9);
  transform: rotate(-45deg);
  z-index: 1;
  pointer-events: none;
}

.search-form input {
  background: rgba(255, 255, 255, 0.18);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 15px;
  color: #fff;
  height: 30px;
  padding: 0 12px 0 30px;
  width: 100%;
  font-size: 0.8rem;
}

.search-form input::placeholder {
  color: rgba(255, 255, 255, 0.9);
  font-size: 0.8rem;
}

.secondary-nav {
  background: #fff;
  border-bottom: 1px solid var(--line);
}

.secondary-nav__inner {
  align-items: center;
  display: flex;
  min-height: 32px;
  overflow-x: auto;
  white-space: nowrap;
}

.secondary-nav a,
.secondary-dropdown {
  align-items: center;
  background: none;
  border: 0;
  color: #171721;
  display: inline-flex;
  font-weight: 400;
  font-size: 0.82rem;
  min-height: 32px;
  padding: 0 11px;
  position: relative;
}

.secondary-nav a[aria-current="page"] {
  color: #171721;
  font-weight: 500;
}

.secondary-nav a[aria-current="page"]::after {
  background: var(--purple-bright);
  bottom: 0;
  content: "";
  height: 4px;
  left: 14px;
  position: absolute;
  right: 14px;
}

.menu-popover {

      @keyframes geapPulse {
        0% { opacity: 0.6; }
        50% { opacity: 0.3; }
        100% { opacity: 0.6; }
      }
    </style>
    <link rel="preload" href="/css/styles.min.css?v=0.1.1" as="style" onload="this.onload=null;this.rel='stylesheet'">
    <noscript><link rel="stylesheet" href="/css/styles.min.css?v=0.1.1"></noscript>
  </head>
  <body>
    <a class="skip-link" href="#app">Skip to content</a>

    <header class="site-header">
      <section class="market-strip" aria-label="Market snapshot">
        <div class="shell market-strip__inner">
          <span><strong>DOW</strong> 51,561.93 <span class="market-badge market-badge--positive">▲ 874.86 (1.73%)</span></span>
          <span><strong>NASDAQ</strong> 26,830.96 <span class="market-badge market-badge--negative">▼ 23.02 (-0.09%)</span></span>
          <span><strong>S&amp;P 500</strong> 7,584.31 <span class="market-badge market-badge--positive">▲ 30.63 (0.41%)</span></span>
          <span class="market-strip__timestamp">as of 10:23 AM ET</span>
          <div class="market-strip__actions">
            <a href="/agent-studio" data-route data-section="geap-admin" class="market-strip__admin"><span class="market-strip__admin-icon">⚙</span> AI Admin</a>
            <span class="market-strip__separator">|</span>
            <a href="#" class="market-strip__logoff" id="etrade-logon-link">Log On</a>
          </div>
        </div>
      </section>

      <section class="utility-bar">
        <div class="shell utility-bar__inner">
          <a class="brand" href="/accounts" data-route aria-label="E*TRADE from Morgan Stanley">
            <img src="LOGO_DATA_URI" alt="E*TRADE from Morgan Stanley" class="brand__logo" width="216" height="57">
          </a>

          <button class="mobile-menu-button" type="button" aria-expanded="false" aria-controls="primary-nav">
            <span></span>
            <span></span>
            <span></span>
            <span class="sr-only">Toggle navigation</span>
          </button>

          <nav class="utility-links" aria-label="Utility navigation">
            <button class="nav-button" type="button" data-menu-button="team">Morgan Stanley Team</button>
            <button class="nav-button" type="button" data-menu-button="support">Support</button>
            <a class="alert-link" href="/alerts" data-route><span class="alert-count">74</span> Alerts</a>
            <a href="/documents" data-route>Documents</a>
            <a href="/profile" data-route>Profile</a>
            <a href="/ai-settings" data-route>AI Settings</a>
          </nav>
        </div>
      </section>

      <nav id="primary-nav" class="primary-nav" aria-label="Primary navigation">
        <div class="shell primary-nav__inner">
          <a href="/accounts" data-route data-section="accounts">Accounts</a>
          <a href="/pay-transfer" data-route data-section="pay-transfer">Pay &amp; Transfer</a>
          <a href="/trading" data-route data-section="trading">Trading</a>
          <a href="/markets-ideas" data-route data-section="markets-ideas">Markets &amp; Ideas</a>
          <a href="/at-work" data-route data-section="at-work">At Work</a>
          <a href="/planning" data-route data-section="planning">Planning</a>
          <a href="/what-we-offer" data-route data-section="what-we-offer">What We Offer</a>
          <a href="/ai-insights" data-route data-section="ai-insights" class="geap-nav-link"><span style="margin-right: 5px;">✦</span>AI Insights</a>


          <form class="search-form" role="search">
            <label class="sr-only" for="site-search">Search</label>
            <input id="site-search" name="q" type="search" placeholder="Search" autocomplete="off">
          </form>
        </div>
      </nav>

      <nav class="secondary-nav" aria-label="Accounts navigation">
        <div class="shell secondary-nav__inner">
          <a href="/accounts" data-route>Complete View</a>
          <a href="/accounts/portfolios" data-route>Portfolios</a>
          <a href="/accounts/watch-lists" data-route>Watch Lists</a>
          <a href="/accounts/orders" data-route>Orders</a>
          <a href="/accounts/balances" data-route>Balances</a>
          <a href="/accounts/activity" data-route>Activity</a>
          <button class="secondary-dropdown" type="button" data-menu-button="banking">Banking</button>
          <a href="/accounts/tax-center" data-route>Tax Center</a>
          <button class="secondary-dropdown" type="button" data-menu-button="documents">Documents</button>
          <a href="/accounts/dividend-reinvestment" data-route>Dividend Reinvestment</a>
          <a href="/accounts/open-account" data-route>Open Account</a>
        </div>
      </nav>

      <nav class="secondary-nav geap-secondary-nav" aria-label="AI Admin navigation" id="geap-secondary-nav" hidden>
        <div class="shell secondary-nav__inner">
          <a href="/agent-studio" data-route>Agent Studio</a>
          <a href="/agent-fabric" data-route>Agent Fabric</a>
        </div>
      </nav>
    </header>

    <div class="menu-popover" data-menu="team" hidden>
      <a href="/accounts" data-route>Advisor contact</a>
      <a href="/planning" data-route>Schedule review</a>
      <a href="/documents" data-route>Shared documents</a>
    </div>
    <div class="menu-popover" data-menu="support" hidden>
      <a href="/support" data-route>Support center</a>
      <a href="/alerts" data-route>Message center</a>
      <a href="/documents" data-route>Forms and documents</a>
    </div>
    <div class="menu-popover" data-menu="banking" hidden>
      <a href="/accounts/banking" data-route>Banking summary</a>
      <a href="/pay-transfer" data-route>Transfer money</a>
      <a href="/accounts/balances" data-route>Balances</a>
    </div>
    <div class="menu-popover" data-menu="documents" hidden>
      <a href="/accounts/documents" data-route>Statements</a>
      <a href="/documents" data-route>Tax documents</a>
      <a href="/accounts/tax-center" data-route>Tax center</a>
    </div>

    <main id="app" tabindex="-1">
      <section class="page-band page-band--gray" style="padding: 24px 0; min-height: 80vh;">
        <div class="shell" style="display: grid; grid-template-columns: 1fr; gap: 20px;">
          <div style="height: 32px; background: #e2e8f0; border-radius: 4px; width: 280px; animation: geapPulse 1.5s infinite ease-in-out;"></div>
          <div style="display: grid; grid-template-columns: 2fr 1fr; gap: 24px;">
            <div style="display: flex; flex-direction: column; gap: 24px;">
              <div style="height: 180px; background: #ffffff; border: 1px solid #e2e8f0; border-radius: 8px; animation: geapPulse 1.5s infinite ease-in-out;"></div>
              <div style="height: 340px; background: #ffffff; border: 1px solid #e2e8f0; border-radius: 8px; animation: geapPulse 1.5s infinite ease-in-out;"></div>
            </div>
            <div style="display: flex; flex-direction: column; gap: 24px;">
              <div style="height: 120px; background: #ffffff; border: 1px solid #e2e8f0; border-radius: 8px; animation: geapPulse 1.5s infinite ease-in-out;"></div>
              <div style="height: 240px; background: #ffffff; border: 1px solid #e2e8f0; border-radius: 8px; animation: geapPulse 1.5s infinite ease-in-out;"></div>
            </div>
          </div>
        </div>
      </section>
    </main>

    <button class="chat-launcher" type="button" aria-expanded="false" aria-controls="agent-panel">
      <span class="chat-launcher__icon" aria-hidden="true"></span>
      Chat
    </button>

    <aside id="agent-panel" class="agent-panel" aria-label="GEAP assistant panel" hidden>
      <div class="agent-panel__header">
        <div>
          <strong>GEAP Assistant</strong>
          <span>Mock data only</span>
        </div>
        <button type="button" class="icon-button" data-close-agent aria-label="Close assistant">x</button>
      </div>
      <div class="agent-panel__body">
        <p>This placeholder is ready for the GEAP agent shell. The next milestone will connect route context, account context, and read-only tools.</p>
        <button type="button" data-agent-chip="summarize">Summarize my accounts</button>
        <button type="button" data-agent-chip="alerts">Triage alerts</button>
        <button type="button" data-agent-chip="portfolio">Explain allocation</button>
      </div>
      <form class="agent-panel__composer">
        <label class="sr-only" for="agent-input">Ask GEAP assistant</label>
        <input id="agent-input" type="text" placeholder="Ask about the mock account view">
        <button type="submit">Send</button>
      </form>
    </aside>

    <script src="/js/data.js?v=0.1.1" defer></script>
    <script src="/js/charts.js?v=0.1.1" defer></script>
    <script src="/js/context.js?v=0.1.1" defer></script>
    <script src="/js/agents.js?v=0.1.1" defer></script>
    <script src="/js/gemini.js?v=0.1.1" defer></script>
    <script src="/js/app.js?v=0.1.1" defer></script>
  </body>
</html>
```


## Appendix H — Markdown renderer (app.js GeapApp.renderMarkdown)




**`public/js/app.js` lines 2135–2160 (verbatim):**

```js
  renderMarkdown(text) {
    if (!text) return "";
    return String(text)
      .replace(/\*\*(.*?)\*\*/g, "<strong>$1</strong>")
      .replace(/\*(.*?)\*/g, "<em>$1</em>")
      .replace(/`(.*?)`/g, "<code>$1</code>")
      .replace(/\[([^\]]+)\]\(([^)]+)\)/g, (match, label, url) => {
        if (url.startsWith("http://") || url.startsWith("https://") || url.startsWith("tel:")) {
          const target = url.startsWith("tel:") ? "" : ' target="_blank" rel="noreferrer"';
          return `<a href="${url}"${target}>${label}</a>`;
        }
        return `<a href="${url}" data-route>${label}</a>`;
      })
      .replace(/\n- /g, "\n• ")
      .replace(/\n/g, "<br>");
  }
};

// Also provide as window.App for backward compatibility
window.App = GeapApp;
```
