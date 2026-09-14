# Emerald TradeFeX

**Predict. Analyze. Trade Smarter.**

AI-powered market intelligence platform — a Bloomberg-terminal-meets-TradingView
experience for probability-based market forecasts, built with Next.js 14 (App
Router), TypeScript, Tailwind CSS, and Supabase.

> AI-generated market analysis. Not financial advice. Markets involve risk.
> Emerald TradeFeX does not execute trades and does not claim guaranteed
> profits, 100% accuracy, or risk-free outcomes.

---

## What's built

- **Landing page** — hero with a live animated forecast panel, scrolling
  price ticker, feature showcase, "how it works", and a track-record teaser
  pulling *real computed* accuracy stats.
- **TradeFeX Command Center** (`/dashboard`) — global market status, AI bias,
  and a live markets grid across forex, commodities, indices, and crypto.
- **AI Predictions** (`/dashboard/predictions`) — the core feature: forecast
  header, candlestick chart (lightweight-charts) with the AI forecast zone
  and invalidation level overlaid, a factor-by-factor "Why TradeFeX AI is
  Bullish" explanation, an animated confidence engine, a trade scenario card,
  and a multi-timeframe consensus matrix.
- **Live prices** — Command Center, AI Predictions, and Watchlist show real
  quotes from Twelve Data where available (forex, crypto), with a
  client-side ticking effect between refreshes, and a graceful per-symbol
  fallback to mock data everywhere else.
- **AI Scanner**, **Market Sentiment** (+ AI News Analyst), **Economic
  Calendar**, **Quant Lab** (backtesting), **Watchlist**, and **AI Track
  Record** pages, all wired to the same service layer.
- A full **Supabase schema** (17 tables, RLS policies, indexes) ready to
  receive real data.
- A **service abstraction layer** so every page reads data through
  `MarketDataService`, `PredictionService`, etc. — never through the mock
  generators directly — so a real data/model provider is a drop-in swap.

## Getting started

```bash
npm install
cp .env.example .env.local
npm run dev
```

The app runs immediately with **zero configuration** using deterministic
mock data (see "Data architecture" below) — no Supabase project or API keys
required to explore the UI.

### Connecting Supabase

1. Create a project at [supabase.com](https://supabase.com).
2. In the SQL Editor, run `supabase/schema.sql`. This creates all 17 tables,
   indexes, and Row Level Security policies, and seeds the `assets` table.
3. Copy your project URL and anon key into `.env.local`:
   ```
   NEXT_PUBLIC_SUPABASE_URL=...
   NEXT_PUBLIC_SUPABASE_ANON_KEY=...
   SUPABASE_SERVICE_ROLE_KEY=...   # server-only, for privileged jobs
   ```
4. Auth (sign-up/sign-in) is wired up: `/auth/login` and `/auth/signup` use
   Supabase Auth directly, and a database trigger creates the matching
   `profiles` row automatically.
5. **Google sign-in**: in the Supabase dashboard, go to Authentication →
   Providers → Google, enable it, and add your Google OAuth Client ID and
   Secret (create these in Google Cloud Console → APIs & Services →
   Credentials). Set the authorized redirect URI to:
   ```
   https://<your-project-ref>.supabase.co/auth/v1/callback
   ```
   No app code changes are needed beyond this — `GoogleAuthButton` already
   calls `supabase.auth.signInWithOAuth({ provider: "google" })`.

## Live prices (Twelve Data)

Prices are no longer purely decorative. The Command Center's market grid,
the AI Predictions page's current-price figure, and the Watchlist all show
a **real quote** wherever the provider can serve one, and fall back to the
mock generator per-symbol otherwise — with a small green dot marking which
ones are live.

1. Create a free account at [twelvedata.com](https://twelvedata.com) and
   copy your API key.
2. Set `MARKET_DATA_API_KEY` in `.env.local` (and in Vercel's environment
   variables for production).
3. That's it — no code changes needed. `lib/services/live/liveMarketData.ts`
   batches all 57 assets into a single request, and `LiveMarketDataService`
   (server-only) merges real prices onto the mock snapshots.

**Coverage**: forex pairs and crypto work reliably on Twelve Data's free
tier. Indices and commodities depend on your plan — any symbol the API
can't serve just quietly keeps its mock price, so nothing ever breaks or
shows a blank state.

**How the "live" feel works**: the server fetches real quotes on a ~20s
interval (batched into one request to stay well inside free-tier rate
limits). Between fetches, a client-side hook (`hooks/useLiveQuotes.ts`)
nudges the displayed price by a tiny, purely cosmetic jitter every ~1.4s so
the UI never looks frozen, then snaps to the next real value when it
arrives.

If `MARKET_DATA_API_KEY` is unset, every price is the mock generator's —
same UI, same behavior, just no green "live" dot.

**A note on scale**: the universe is now 57 assets. Twelve Data's free
tier has a fairly low per-minute request-credit limit, and each symbol in
a batched quote counts against it — so on the free plan, a single refresh
covering all 57 symbols may get rate-limited and fall back to mock for
that cycle (this is handled gracefully; nothing breaks, you just see fewer
live dots). A paid Twelve Data plan raises that ceiling. If you want live
coverage guaranteed regardless of plan, the fix is to fetch in smaller
chunks across multiple intervals rather than one batch — not implemented
here, flagged as a next step.

## Data architecture

```
UI components
   │
   ▼
Service layer (lib/services/*.ts)      ← pages only ever import from here
   │
   ├── mock: lib/services/mock/*
   │     deterministic, seeded generators — same inputs always produce
   │     the same output within a day, so the demo never flickers or
   │     hydration-mismatches between server and client renders
   │     (predictions, sentiment, news, calendar, scanner, track record)
   │
   └── live: lib/services/live/*  ← implemented for market prices today
         liveMarketData.ts calls Twelve Data server-side; everything else
         (predictions, sentiment, news, calendar) is still mock-only —
         follow the same pattern to add a live provider for those
```

`lib/services/marketDataService.ts` only ever returns mock data and is safe
to import from client components. The live merge lives in a separate file,
`lib/services/marketDataService.server.ts` (`LiveMarketDataService`), which
imports `lib/services/live/liveMarketData.ts` — a `"server-only"`-guarded
module. **This split is deliberate**: if the live provider's code got
pulled into a client bundle (e.g. via a shared import), Next.js's
`server-only` package throws a build error rather than silently shipping a
server-side fetch pattern (or, worse, a credential) to the browser. Only
call `LiveMarketDataService` from a Server Component or Route Handler.

To connect a real provider for anything else (predictions, sentiment, news,
economic calendar):

1. Implement the same exported function signatures in a new file under
   `lib/services/live/` (e.g. `livePredictionProvider.ts`), with
   `import "server-only"` at the top.
2. Do the actual network/model call server-side only.
3. Either call it directly from a server-only wrapper file (see
   `marketDataService.server.ts` for the pattern) if any client component
   imports the corresponding `*Service.ts`, or call it straight from the
   `*Service.ts` file if nothing client-side imports that one.

**The AI prediction engine** (`lib/services/mock/mockPredictionProvider.ts`)
is intentionally structured so the explanation text, confidence breakdown,
and trade scenario are all *derived from the same factor object* — swapping
in a real model just means that object comes from your model's output
instead of a seeded random generator; nothing downstream changes.

**The AI Track Record** (`lib/services/trackRecordService.ts`) always
computes accuracy, drawdown, and P/L simulation from an array of outcome
rows — it never hardcodes a percentage. In production, back that array with
a query against the `prediction_outcomes` table.

## Project structure

```
app/
  page.tsx                    Landing page
  dashboard/
    layout.tsx                Authenticated shell (nav)
    page.tsx                  Command Center
    predictions/page.tsx      AI Predictions
    scanner/page.tsx          AI Scanner
    sentiment/page.tsx        Market Sentiment + AI News Analyst
    calendar/page.tsx         Economic Calendar
    backtesting/page.tsx      Quant Lab
    watchlist/page.tsx        My Markets
    track-record/page.tsx     AI Track Record
components/
  nav/                        TopNav, MobileNav
  ui/                         Card, BiasBadge, ConfidenceRing, StatBar, SymbolSelect
  markets/                    AssetCard
  predictions/                ForecastChart, WhyTradeFexAI, ConfidenceEngine,
                               ScenarioCard, MultiTimeframeMatrix, SentimentBars
  landing/                    HeroForecastPanel, LiveTicker, FeatureShowcase,
                               HowItWorks, TrackRecordTeaser, Footer
lib/
  types.ts                    Shared domain types (the service contract)
  services/                   Abstraction layer (Market/Prediction/Sentiment/
                               News/EconomicCalendar/Scanner/Backtest/TrackRecord)
  services/mock/               Deterministic mock providers
  supabase/                   Browser, server, and service-role clients
supabase/
  schema.sql                  All 17 tables + RLS + indexes + asset seed data
```

## Design system

- **Palette**: near-black base (`#0A0C0B`), graphite surfaces (`#121512`),
  emerald primary (`#1FAE71`), deep emerald, muted electric cyan (`#4FD1C5`)
  and subtle gold (`#C9A65B`) as sparing accents, muted red for bearish
  states (`#D9605C`) — deliberately not a neon crypto palette.
- **Type**: Manrope for display/headings, Inter for body and UI, with
  tabular numerals (`.font-nums`) on every financial figure so digits align.
- **Motion**: used only where it clarifies state — animated confidence rings
  and bars, a single hero chart reveal, a slow ticker — not on every card.

## Notes on scope

This is a complete, runnable architecture with polished core screens
(landing, Command Center, AI Predictions, Auth). Not yet implemented, as
natural next steps:

- A live news/economic-calendar provider (currently mock-only — market
  prices are live via Twelve Data, see above).
- The "TradeFeX Intelligence" conversational assistant and the "What If?"
  scenario tool (would call an LLM server-side with the same Prediction/
  Sentiment/News objects already modeled here as context).
- A cron/edge function to resolve predictions into `prediction_outcomes`
  rows on a schedule, which is what the Track Record page is designed to
  read from in production.
- Per-user, persisted alerts (the Alert Center currently shows a shared
  deterministic feed; the `alerts` table and RLS are ready for real rows).
