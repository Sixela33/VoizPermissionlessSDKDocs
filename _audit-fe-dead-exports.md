# Audit: FE `@voiz/markets-sdk` imports + dead public API

Read-only. Snapshot: 2026-09-02 (America/Buenos_Aires).  
SDK `src/index.ts` / `package.json` already partially trimmed (no `./ponder` export, no `*Calls` / raw ABI barrel / `useVoizClient`).  
**Do not change SDK source here** — checklist only.

---

## Summary counts

| Metric | Count |
|--------|------:|
| FE files importing `@voiz/markets-sdk` (+ `/react`) | 25 |
| Distinct FE-imported **value** symbols | 12 |
| Distinct FE-imported **type** symbols | 10 |
| Public symbols in `src/index.ts` | 58 |
| Public symbols in `src/react/index.ts` | 6 |
| `package.json` export paths | 2 (`.` , `./react`) — `./ponder` already gone |
| Main/react exports with **zero** FE + zero smoke named importers | **43** |
| FE bypasses of `useVoiz().client` (flagged) | 9 patterns |
| Dist vs src barrel symbol drift | **0** (main + react barrels match) |
| Dist orphans / weight | `dist/ponder/**` still built (24 files); smoke deep-imports `dist/actions/*`, `dist/abis/*` |

---

## 1) Every FE `@voiz/markets-sdk` import (symbol → file)

### Values — `@voiz/markets-sdk`

| Symbol | Files | Flag |
|--------|-------|------|
| `VoizMarketsClient` | `lib/fetch-market.ts`, `app/api/market-image/route.ts` | **bypass** — `new VoizMarketsClient(...)` outside provider (server/lib OK; not `useVoiz().client`) |
| `erc20Abi` | `app/hooks/useDildoBalance.ts`, `app/profile/page.tsx`, `app/transfer/page.tsx`, `components/FaucetButton.tsx` | **bypass** — raw ABI for viem `readContract` / writes |
| `getAddresses` | `lib/chain.ts`, `components/FaucetButton.tsx` | **bypass** — prefer `useVoiz().client.addresses` in UI |
| `normalizeOddsBps` | `app/create/page.tsx`, `components/markets/LiquidityPanel.tsx` | **bypass** — math (UI); not via client |
| `previewSeedTotals` | `app/create/page.tsx`, `components/markets/SeedFields.tsx` | **bypass** — math (UI) |
| `aggregateOutcomeCollateral` | `components/markets/MarketCard.tsx`, `lib/market-summary-stats.test.ts` | **bypass** — ponder/display helper |
| `buildMarketOutcomeStats` | `lib/market-summary-stats.test.ts` only | **bypass** — test-only |
| `selectNewestPriceByOutcome` | `lib/market-summary-stats.test.ts` only | **bypass** — test-only |
| `marketImagePath` | `app/api/market-image/route.ts`, `lib/market-image-url.ts` | mild — path util (fine for API/SSR) |
| `resolvedDisplayRows` | `lib/resolved-display.ts` (re-export) | mild — display helper |

### Types — `@voiz/markets-sdk`

| Symbol | Files |
|--------|-------|
| `MarketSummary` | `app/hooks/useMarkets.ts`, `useResolverAccess.ts`, `lib/fetch-market.ts`, `lib/holdings.ts`, `components/markets/{InvestPanel,LiquidityPanel,MarketCard,MarketImageEditor,PriceHistoryChart,ResolvePanel}.tsx` |
| `VoizMarketsClient` (type) | `app/create/page.tsx`, `lib/holdings.ts` |
| `Call` | `app/providers/VoizProviderBridge.tsx` |
| `ResolverKind` | `app/create/page.tsx` |
| `LpPosition` | `components/markets/LiquidityPanel.tsx` |
| `AllMarketsLpPosition` | `app/profile/page.tsx` |
| `UserSharePosition` | `lib/holdings.ts` |
| `PonderTrade` | `lib/holdings.ts` |
| `PonderPricePoint` | `lib/price-history.ts` |

### `@voiz/markets-sdk/react`

| Symbol | Files | Flag |
|--------|-------|------|
| `useVoiz` | `app/create/page.tsx`, `app/hooks/{useMarkets,useCollateralBalance,useResolverAccess}.ts`, `app/markets/[address]/page.tsx`, `app/profile/page.tsx`, `components/markets/{InvestPanel,LiquidityPanel,PriceHistoryChart,ResolvePanel}.tsx` | **KEEP** — canonical |
| `VoizProvider` | `app/providers/VoizProviderBridge.tsx` | **KEEP** |
| `VoizSendCalls` | `app/providers/VoizProviderBridge.tsx` | **KEEP** |

### Not found in FE (good)

- No `*Calls` / encode helpers / raw `ponderGraphql` / `fetch*FromPonder`
- No ABIs other than `erc20Abi`
- No `useVoizClient` (already removed from `src/react`)
- No `@voiz/markets-sdk/ponder` subpath imports

Protocol UI that **does** go through client (evidence): create/seed/swap/resolve/burn/merge/LP/`listMarkets`/`summary` via `useVoiz().client` in create, InvestPanel, LiquidityPanel, ResolvePanel, useMarkets, profile, etc.

---

## 2) Dead public API checklist (zero FE importers, zero smoke named importers from `./dist/index.js`)

Legend: **remove** = safe to drop from public barrel; **keep** = used or part of intended library surface; **migrate** = FE should stop importing / move behind client.

### `src/index.ts` — KEEP (FE uses)

- [ ] **keep** `VoizMarketsClient` — FE + smoke
- [ ] **keep** `getAddresses` — FE (+ smoke); see migrate
- [ ] **keep** `normalizeOddsBps`, `previewSeedTotals` — FE UI math
- [ ] **keep** `resolvedDisplayRows`, `erc20Abi`, `marketImagePath`
- [ ] **keep** `aggregateOutcomeCollateral` — FE MarketCard
- [ ] **keep** `buildMarketOutcomeStats`, `selectNewestPriceByOutcome` — FE test only (or demote to test util)
- [ ] **keep** types: `Call`, `ResolverKind`, `LpPosition`, `MarketSummary`, `PonderPricePoint`, `PonderTrade`, `AllMarketsLpPosition`, `UserSharePosition`

### `src/index.ts` — REMOVE candidates (no external importers)

**Values**

- [ ] **remove** `registerAddresses`
- [ ] **remove** `DEFAULT_CHAIN_ID`
- [ ] **remove** `baseSepolia` (smoke uses `viem/chains`, not SDK)
- [ ] **remove** `base`
- [ ] **remove** `VoizPool` — no direct import; obtained via `client.pool()`
- [ ] **remove** `VoizOutcome` — via `client` / market
- [ ] **remove** `VoizMarket` — via `client.market()`
- [ ] **remove** `VoizUser` — via `client.user()`

> Note: removing class re-exports does not delete implementations; only public named exports. Advanced consumers typing `VoizMarket` would need `ReturnType` / inferred types.

**Types (param / internal result shapes unused by FE)**

- [ ] **remove** `CreateMarketParams`, `CreateMarketArgs`, `SeedLeg`, `SeedPlan`, `V4PoolKey`
- [ ] **remove** `ChainAddresses`, `AddressOverrides`, `ClientConfig`
- [ ] **remove** `SendCalls`, `SendCallsOptions` (root) — FE uses react `VoizSendCalls`
- [ ] **remove** `SeedMarketParams`, `SeedOutcomeParams`, `RedeemParams`, `AddLiquidityParams`, `MarketSnapshot`
- [ ] **remove** `LpPositionInput`, `ListMarketsResult`
- [ ] **remove** `MarketOutcomeStats`, `IndexedOutcomeStats`, `IndexedPricePoint`
- [ ] **remove** `PoolStatus`, `PoolState`, `QuoteExactInResult`
- [ ] **remove** `OutcomePriceRow`, `OutcomeMetricsRow`, `DisplayOutcomePrices`, `DisplayOutcomeMetrics`
- [ ] **remove** `EncodeResolveParams`, `EncodeMergeParams`
- [ ] **remove** `UserLpPositionsParams`, `UserTradesParams`, `UserPositionsParams`

### `src/react/index.ts`

- [ ] **keep** `VoizProvider`, `useVoiz`, `VoizSendCalls`
- [ ] **keep** `VoizProviderProps`, `VoizContextValue`, `SendCallsOptions` — no FE `import type`, but they are the provider’s public typing surface (not “dead” in the same sense as unused root types)

### `package.json` `exports`

- [ ] **keep** `.` and `./react`
- [x] **remove** `./ponder` — **already done** (no FE importers; only mentioned in sdkdocs)

---

## 3) FE migrate checklist (bypass → client)

- [ ] **migrate** `getAddresses` in `lib/chain.ts` / `FaucetButton.tsx` → `useVoiz().client.addresses` (or inject addresses from provider) where React tree allows
- [ ] **migrate** `erc20Abi` usages → `client.getErc20Balance` / client write helpers where possible (`profile` already mixes both)
- [ ] **migrate** (optional) `aggregateOutcomeCollateral` on `MarketCard` → field on `MarketSummary` or `client` helper so FE drops ponder helper import
- [ ] **migrate** (optional) seed math: expose `client.previewSeed…` if product wants zero math imports
- [ ] **keep** server `new VoizMarketsClient` in `lib/fetch-market.ts` + `app/api/market-image/route.ts` (non-React)

---

## 4) Dist vs src drift

| Check | Status |
|-------|--------|
| `src/index.ts` ↔ `dist/index.d.ts` symbol set | **Match** (58) |
| `src/react/index.ts` ↔ `dist/react/index.d.ts` | **Match** (6); `useVoizClient` / `hooks.ts` gone from src+dist |
| `package.json` exports ↔ reality | OK (`.` + `./react`) |
| `dist/ponder/**` | Still emitted (24 files) though **not** a package export — publish weight / deep-import risk |
| Smoke vs barrel | **`scripts-check-liq.mjs`** imports `planMarketSeed` from `./dist/index.js` but it is **not** exported anymore; also deep-imports `./dist/actions/seedMarket.js`. `quote-decode.mjs` deep-imports `./dist/abis/uniswap.js`. `smoke-gateoff-redeploy.mjs` deep-imports `./dist/actions/createMarket.js` |

---

## Top 10 highest-value removals

1. **`registerAddresses` + `DEFAULT_CHAIN_ID` + `base` + `baseSepolia`** — unused public chain surface (smoke already uses `viem/chains`).
2. **`VoizPool` / `VoizOutcome` / `VoizMarket` / `VoizUser` value re-exports** — zero importers; FE uses `client.*` factories only.
3. **Root `SendCalls` / `SendCallsOptions`** — FE wired on react `VoizSendCalls`; duplicate surface.
4. **Create/seed/redeem/LP param types** (`CreateMarketParams`, `SeedMarketParams`, `RedeemParams`, `AddLiquidityParams`, …) — zero FE importers; keep internal to `client` methods.
5. **Pool/quote/display type cluster** (`PoolState`, `QuoteExactInResult`, `DisplayOutcome*`, `EncodeResolveParams`, …) — unused outside SDK.
6. **`MarketOutcomeStats` + `IndexedOutcomeStats` + `IndexedPricePoint` + `ListMarketsResult` + `LpPositionInput`** — unused types.
7. **`User*Params` types** (`UserLpPositionsParams`, `UserTradesParams`, `UserPositionsParams`) — unused.
8. **Stop shipping / stop relying on non-exported dist deep paths** — fix smoke to use client API; then `dist/actions/*` + most `dist/abis/*` need not be treated as public.
9. **`dist/ponder/**` publish weight** — not in `exports`; either don’t emit for publish or accept as internal-only (root already re-exports the 4 helpers FE needs).
10. **FE-only: demote test helpers** — `buildMarketOutcomeStats` + `selectNewestPriceByOutcome` only used in `lib/market-summary-stats.test.ts`; candidates to drop from public API if tests import from relative SDK paths or assert via `MarketSummary` shape.

---

## Evidence roots

- FE: `/home/s1x3l4/Desktop/voiz/permissionlessMarkets/frontend`
- SDK: `/home/s1x3l4/Desktop/voiz/permissionlessMarkets/sdkDev/sdk`
- Barrels: `src/index.ts`, `src/react/index.ts`, `package.json` `exports`
- Smoke breakage: `sdkDev/sdk/scripts-check-liq.mjs` (`planMarketSeed`)
