# CoinInfo Calendar

A financial event calendar embedded in a DEX trading interface. Built as a side-panel drawer alongside a candlestick chart, designed for traders who need to monitor macro events while managing perpetual contract positions.

Live: https://coininfo-tawny.vercel.app/

---

## What it does

Opens as a resizable right-side drawer (TradingView-style). Shows upcoming economic, earnings, crypto, and IPO events in a weekly view with day cards, date navigation, and per-symbol context (Next Event card + Market Consensus table pinned above the calendar).

The panel is symbol-aware: switching between BTC-PERP, TSLA-PERP, WTI-PERP resets the calendar state and surfaces the most relevant upcoming event for that asset.

---

## 4-Tab Information Architecture

Simplified to ≤ 3 event types per category. Every event shows **Actual / Forecast / Prior** where applicable.

### Economic
*Macro data that moves commodity and crypto prices*

| Event type | Key fields |
|---|---|
| Central bank rate decision | Country · Rate · A/F/P · Time (UTC) |
| CPI / Inflation | Country · Index · A/F/P · MoM or YoY |
| EIA Crude Inventories | Weekly change · A/F/P · Unit (M bbl) |

Impact level color-coded: 🔴 High · 🟡 Med · ⚫ Low

### Earnings
*US and KR stock earnings relevant to perp pairs (e.g. TSLA-PERP)*

| Event type | Key fields |
|---|---|
| EPS (GAAP) | Ticker · Company · A/F/P · Surprise % |
| Revenue | Ticker · Company · A/F/P |
| Release timing | BMO (before open) or AMC (after close) badge |

### Crypto
*On-chain events that directly affect DEX asset prices*

| Event type | Key fields |
|---|---|
| Token unlock | Symbol · Amount · USD value · % of supply |
| ETF net flow | Direction (inflow/outflow) · Daily USD amount · Cumulative |
| Major upgrade / hard fork | Chain · Event name · Block height or date |

### IPO
*NASDAQ listings only — used as a risk-appetite signal for crypto markets*

| Event type | Key fields |
|---|---|
| New listing | Company · TICKER · Exchange (NASDAQ) |
| Roadshow / Pricing date | Date window |
| Deal size | Price range · Raise amount ($M) |

---

## Implementation Plan

### Data layer

Each event conforms to a shared schema:

```js
{
  id: string,
  date: 'YYYY-MM-DD',
  timeUtc: 'HH:MM',            // null for all-day events
  tab: 'economic' | 'earnings' | 'crypto' | 'ipo',
  market: 'crypto' | 'us-stock' | 'kr-stock' | 'commodities' | 'nasdaq',
  symbol: string | null,        // e.g. 'BTC', 'TSLA', 'WTI' — for symbol-aware filtering
  impact: 'high' | 'med' | 'low',
  title: string,
  country: string | null,       // emoji flag or null
  actual: string | null,
  forecast: string | null,
  prior: string | null,
  unit: string | null,          // '%', 'M bbl', '$B', etc.
  detail: string | null,        // expanded row description
  source: string | null,        // URL
  // Earnings-specific
  epsSurprise: number | null,
  timing: 'BMO' | 'AMC' | null,
  // Crypto-specific
  unlockAmount: string | null,
  unlockUsd: string | null,
  supplyPct: string | null,
  // IPO-specific
  priceRange: string | null,
  raiseUsd: string | null,
}
```

**Data sources (future integration):**
- Economic: TradingEconomics API or Forex Factory feed
- Earnings: Financial Modeling Prep / Alpha Vantage
- Crypto unlocks: Token Unlocks API / Defillama
- ETF flows: CoinGlass or Farside Investors
- NASDAQ IPO: SEC EDGAR EDGAR filings or Renaissance Capital feed

### Frontend components

```
CalendarDrawer (fixed right panel, resizable)
├── DrawerHeader            — "Calendar" title + close button
├── NextEventCard           — pinned; symbol-aware; collapsed/expanded states
├── MarketConsensusTable    — pinned; A/F/P rows for current symbol's next event
├── ScrollRegion
│   ├── Toolbar             — Today · date picker · week nav · market filter · TZ
│   ├── WeekCards           — 7 day cards with per-tab event counts + swipe
│   ├── TabBar              — Economic · Earnings · Crypto · IPO
│   └── EventList           — grouped by date; expandable rows; tab-switched renderer
└── ResizeHandle            — left-edge drag grip (desktop only)
```

### Tab renderers

Each tab uses the same **date-grouped row container** but a different column layout:

| Tab | Columns |
|---|---|
| Economic | `[flag+name 1fr] [Actual 64px] [Forecast 80px] [Prior 64px]` |
| Earnings | `[ticker+company 1fr] [EPS A/F/P] [Rev A/F/P] [BMO/AMC badge]` |
| Crypto | `[symbol+event 1fr] [Amount] [USD value] [Supply %]` |
| IPO | `[company+ticker 1fr] [Price range] [Raise $M] [Date]` |

### Symbol-aware filtering

`currentAsset` (e.g. `BTC-PERP`) maps to an asset key used to:
1. Pin the Next Event card to the most relevant upcoming event
2. Pre-filter the Crypto tab when the symbol matches a known token
3. Show the Market Consensus table rows specific to that asset

Switching symbol calls `resetCalendarState()` which resets week offset, clears selection, and re-renders the pinned cards.

### State

```js
calWeekOffset     // integer weeks from today
calSelectedTs     // null or selected date timestamp
calMarketFilter   // 'all' | 'crypto' | 'us-stock' | 'kr-stock' | 'commodities' | 'nasdaq'
calActiveTab      // 'Economic' | 'Earnings' | 'Crypto' | 'IPO'
calTzOffset       // integer UTC offset, defaults to local timezone
currentAsset      // e.g. 'BTC-PERP'
```

### Week card event counts

Each day card shows 4 colored dot-counts (one per tab) so traders see at a glance which days are event-heavy:

```
Mon 15
 15
● 3  ● 7  ●crypto 2  ●ipo 1
```

Colors: blue (Economic) · violet (Earnings) · orange (Crypto) · green (IPO)

---

## Tech stack

- Single HTML file — no build step, no framework
- Tailwind CSS (Play CDN)
- Lucide icons (CDN)
- Deployed on Vercel (auto-deploy from `main`)
- Playwright for regression testing (local, not in CI)

---

## Local dev

```bash
cd coininfo
python3 -m http.server 8123
# open http://localhost:8123/index.html
```

Tests (requires Playwright):
```bash
cd /tmp
node test_calnav.js
node test_calnav2.js
node verify_items.js
node verify_p5.js
```

Source of truth is `/Users/a888/Downloads/dipcoin-preview/88.html` — synced to this repo's `index.html` before each commit.
