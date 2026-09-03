# Audit: public `@voiz/markets-sdk` surface — what should leave

**Mode:** read-only (no SDK/FE source edits).  
**Snapshot:** 2026-09-02 ~22:43 America/Argentina/Buenos_Aires (UTC-3).  
**Sources:** `sdk/src/index.ts`, `sdk/src/react/index.ts`, public class methods on `VoizMarketsClient` / `VoizMarket` / `VoizPool` / `VoizUser` vs FE imports + `client.*` / `market.*` / `pool.*` / `user.*` call sites under `frontend/`.  
**Principle:** less public surface > more helpers. Prefer **privatize/delete** over keep. Docs/smoke deep-imports do not justify barrel exports.

Related prior notes (partially stale after barrel trims): `_audit-fe-dead-exports.md`, `_audit-duplicate-apis.md`, `_audit-react-types-abi.md`.

---

## Executive summary

| Bucket | Count (approx) | Action bias |
|--------|---------------:|-------------|
| Root barrel symbols (`src/index.ts`) | 42 | Cut ~25 type/value exports FE never names |
| React barrel (`src/react/index.ts`) | 6 | **Keep** (FE uses provider + `useVoiz`) |
| Client public methods unused by FE | 3+ | Privatize RPC fallback / submit / `withAddresses` |
| Market public methods unused by FE | 3+ | Privatize `outcomePrices`, `listLpPositions`, `outcome` |
| Pool public surface leaks / duals | several | Hide `key`, fold `isPriceAboveNinetyNineCents`, demote mid helpers |
| Dual catalog APIs | `listMarkets` vs `listFactoryMarkets` vs `listMarketSummariesFromRpc` | One catalog + privatize RPC sibling |

FE already goes through `useVoiz().client` for protocol UI. Remaining root imports are a thin set of math/display helpers + types + `erc20Abi` / `getAddresses`.

---

## FE named imports (ground truth)

### Values — `@voiz/markets-sdk`

| Symbol | FE sites |
|--------|----------|
| `VoizMarketsClient` | `lib/fetch-market.ts`, `app/api/market-image/route.ts` (+ type in create/holdings) |
| `getAddresses` | `lib/chain.ts`, `FaucetButton.tsx` |
| `normalizeOddsBps` | create, LiquidityPanel |
| `quoteSeed` | create, SeedFields |
| `erc20Abi` | useDildoBalance, profile, transfer, FaucetButton |
| `aggregateOutcomeCollateral` | MarketCard (+ test) |
| `marketImagePath` | market-image route, `lib/market-image-url.ts` |
| `resolvedDisplayRows` | only via `lib/resolved-display.ts` re-export (+ its test) |

### Types — `@voiz/markets-sdk`

`MarketSummary`, `Call`, `ResolverKind`, `LpPosition`, `AllMarketsLpPosition`, `UserSharePosition`, `PonderTrade`, `PonderPricePoint` (+ `VoizMarketsClient` as type).

### `@voiz/markets-sdk/react`

`useVoiz`, `VoizProvider`, `VoizSendCalls` — **Keep**. Props/context option types are the provider’s typing surface even without direct `import type`.

### Class methods FE actually calls

| Owner | Used |
|-------|------|
| **Client** | `addresses`, `market`, `pool`, `user`, `listMarkets`, `listFactoryMarkets`, `isMarket`, `createMarket`, `redeem`, `burn`, `getErc20Balance`, `getCollateralBalance` |
| **Market** | `summary`, `seed`, `seedStatus`, `displayPrices`, `displayMetrics`, `priceHistory`, `resolve`, `merge`, `addLiquidityAcrossOutcomes`, `canResolve` (+ field reads `wrappedTokens` / create flows) |
| **Pool** | `state`, `priceFromState`, `quoteTrade`, `swap`, `estimatePriceAfterSwap`, `isPriceAboveNinetyNineCents` |
| **User** | `trades`, `positions`, `lpPositions` |

---

## Ranked: Privatize / Delete / Keep

Impact = (API confusion or leak severity) × (ease of removal) × (FE blast radius inverted). Higher = cut sooner.

### P1 — Privatize / delete first (high impact, low FE risk)

| # | Item | Verdict | Why | FE blast |
|---|------|---------|-----|----------|
| 1 | **`client.listMarketSummariesFromRpc`** | **Privatize** (`private` or unexported) | Still public; FE never calls. Only used inside `listMarkets` fallback. Publishing it invites a second catalog API. | None |
| 2 | **`VoizMarket.outcomePrices`** | **Privatize** | Dual of **`displayPrices`** (FE uses only display). Already the RPC fallback inside `displayPrices`. Advertising both teaches apps to skip indexer. | None (no `.outcomePrices` in FE) |
| 3 | **Root `V4PoolKey` + `VoizPool.key` + `LpPosition.poolKey`** | **Privatize / strip from DTO** | Classic V4 leak. FE burn uses `outcomeToken` only; LiquidityPanel never reads `.poolKey`. `client.burn` already builds the key internally. Keep `poolKey` module-private; drop field from public `LpPosition` (or `@internal` + omit from docs). | None today; regenerate types only |
| 4 | **Root `EncodeResolveParams` / `EncodeMergeParams`** | **Delete from barrel** | Encode* leakage; encode paths are already private on `VoizMarket`. FE never imports. | None |
| 5 | **`VoizMarket.listLpPositions`** | **Privatize** | Enrich-only helper for `user.lpPositions`. FE never calls. Parallel “list LP” API vs User. | None |
| 6 | **Root unused param/result types** | **Delete from barrel** | FE never names: `CreateMarketParams`, `CreateMarketArgs`, `SeedLeg`, `SeedPlan`, `SeedMarketParams`, `SeedOutcomeParams`, `RedeemParams`, `LpPositionInput`, `ListMarketsResult`, `ChainAddresses`, `AddressOverrides`, `ClientConfig`, `PoolStatus`, `PoolState`, `QuoteExactInResult`, `OutcomePriceRow`, `OutcomeMetricsRow`, `DisplayOutcomePrices`, `DisplayOutcomeMetrics`, `UserLpPositionsParams`, `UserTradesParams`, `UserPositionsParams`, `MarketOutcomeStats`. Apps can infer from methods; exporting them freezes a wide contract. | None (inference still works) |

### P2 — Dual APIs / method demotions (medium impact; small FE migrate)

| # | Item | Verdict | Why | FE blast |
|---|------|---------|-----|----------|
| 7 | **`listFactoryMarkets` vs `listMarkets`** | **Keep `listMarkets` public; narrow `listFactoryMarkets`** | Catalog = `listMarkets`. Factory address list is still used by `lib/fetch-market.ts` + create “detect new market” poll. Prefer: privatize later after FE uses `listMarkets().markets.map(m => m.address)` or a tiny `listFactoryAddresses()` marked advanced — or keep **one** address-list method but stop documenting it next to catalog. | Low–medium if renamed |
| 8 | **`pool.isPriceAboveNinetyNineCents`** | **Privatize** (after FE migrate) | Duplicate of `quoteTrade` → `priceAboveNinetyNineCents`. InvestPanel already prefers quote field and only falls back to the method. Fold gate into quote/`state` consumers. | 1 file (`InvestPanel`) |
| 9 | **`pool.estimatePriceAfterSwap`** | **Privatize** (after FE migrate) | Overlaps `quoteTrade.priceAfter`. InvestPanel still calls it for preview when quote path differs — migrate to quote enrichment only. | 1 file |
| 10 | **`client.submitCalls`** | **Keep `@internal` / make `/** @internal */` + strip from public `.d.ts` if possible** | Marked internal; still a public class method. FE never calls. True consumers are Market/Pool/Client send paths. | None |
| 11 | **`client.withAddresses`** | **Privatize or delete** | Zero FE usage. Nice-to-have; not needed for product surface. | None |
| 12 | **`pool.outcomePrice` / `collateralReserve`** | **Privatize** | Only used inside market display/metrics fallbacks. FE never calls. | None |
| 13 | **`VoizMarket.outcome` / `VoizOutcome`** | **Privatize / stop advertising** | No FE `.outcome(` usage. Facade adds surface without product calls. | None |

### P3 — Root helpers FE still imports (migrate then delete)

| Item | Verdict | Why |
|------|---------|-----|
| `resolvedDisplayRows` | **Privatize** after FE drops re-export | Only `lib/resolved-display.ts` + test; resolved UI should use `displayPrices` / summary. |
| `aggregateOutcomeCollateral` | **Keep short-term** or fold into `MarketSummary` | MarketCard depends; better as summary field than ponder helper export. |
| `normalizeOddsBps` / `quoteSeed` | **Keep** (UI math) | Create + seed UX; acceptable small public math surface. |
| `marketImagePath` | **Keep** | SSR/API path util; fine. |
| `getAddresses` / `erc20Abi` | **Keep** (or migrate UI to `client.addresses` / `getErc20Balance`) | Non-React + faucet paths still need them. |
| `PonderTrade` / `PonderPricePoint` | **Keep** (or rename without `Ponder*` prefix) | FE holdings/price-history types. Prefix leaks indexer; prefer `Trade` / `PricePoint` later. |
| `MarketSummary`, `LpPosition`, `AllMarketsLpPosition`, `UserSharePosition`, `Call`, `ResolverKind` | **Keep** | Real FE DTOs. Strip `poolKey` from `LpPosition` (see P1#3). |

### Keep (do not cut)

| Item | Why |
|------|-----|
| `VoizMarketsClient` + factories `market` / `pool` / `user` | Canonical entry |
| `listMarkets`, `createMarket`, `redeem`, `burn`, balances | FE product path |
| `market.summary` / `seed` / `seedStatus` / `displayPrices` / `displayMetrics` / `priceHistory` / `resolve` / `merge` / `addLiquidityAcrossOutcomes` / `canResolve` | FE product path |
| `pool.state` / `priceFromState` / `quoteTrade` / `swap` | InvestPanel |
| `user.trades` / `positions` / `lpPositions` | holdings / profile / LP panel |
| `isMarket` | `fetch-market` factory gate |
| React: `VoizProvider`, `useVoiz`, `VoizSendCalls` (+ props/context types) | Canonical React binding |

---

## Dual-API callouts (explicit)

| Pair | Canonical | Leave / hide |
|------|-----------|--------------|
| `displayPrices` vs `outcomePrices` | `displayPrices` | Privatize `outcomePrices` |
| `listMarkets` vs `listFactoryMarkets` vs `listMarketSummariesFromRpc` | `listMarkets` | Privatize `listMarketSummariesFromRpc`; demote factory list |
| `quoteTrade.priceAfter` vs `estimatePriceAfterSwap` | `quoteTrade` | Privatize standalone after FE migrate |
| `quoteTrade.priceAboveNinetyNineCents` vs `isPriceAboveNinetyNineCents` | quote field | Privatize method after FE migrate |
| `user.lpPositions` vs `market.listLpPositions` | `user.lpPositions` | Privatize market enrich |
| `client.burn({ outcomeToken })` vs passing `poolKey` | outcomeToken API | Remove `LpPosition.poolKey` / `pool.key` from public |

---

## Barrel checklist (current `src/index.ts`)

**Keep (FE or intentional):**  
`VoizMarketsClient`, `getAddresses`, `normalizeOddsBps`, `quoteSeed`, `resolvedDisplayRows` *(until FE migrate)*, `erc20Abi`, `marketImagePath`, `aggregateOutcomeCollateral`, types: `Call`, `ResolverKind`, `LpPosition` *(sans poolKey)*, `AllMarketsLpPosition`, `UserSharePosition`, `MarketSummary`, `PonderPricePoint`, `PonderTrade`.

**Delete from barrel (FE = 0 named imports):**  
`ListMarketsResult`, `CreateMarketParams`, `CreateMarketArgs`, `SeedLeg`, `SeedPlan`, `V4PoolKey`, `ChainAddresses`, `AddressOverrides`, `ClientConfig`, `SeedMarketParams`, `SeedOutcomeParams`, `RedeemParams`, `LpPositionInput`, `PoolStatus`, `PoolState`, `QuoteExactInResult`, `OutcomePriceRow`, `OutcomeMetricsRow`, `DisplayOutcomePrices`, `DisplayOutcomeMetrics`, `EncodeResolveParams`, `EncodeMergeParams`, `UserLpPositionsParams`, `UserTradesParams`, `UserPositionsParams`, `MarketOutcomeStats`.

**React barrel:** keep as-is.

---

## Top 10 actionable cuts

1. **Privatize `listMarketSummariesFromRpc`** — stop a second public catalog; keep as `listMarkets` implementation detail.  
2. **Privatize `VoizMarket.outcomePrices`** — single display API: `displayPrices` only.  
3. **Remove public `V4PoolKey` / `pool.key` / `LpPosition.poolKey`** — end Uniswap key leakage; burn already takes `outcomeToken`.  
4. **Un-export `EncodeResolveParams` + `EncodeMergeParams`** — encode types must not be public.  
5. **Privatize `VoizMarket.listLpPositions`** — LP discovery stays on `user.lpPositions`.  
6. **Strip ~20 unused root types** from `src/index.ts` (Create*/Seed*/Redeem*/Pool*/Display*/User*Params/`MarketOutcomeStats`/`ListMarketsResult`/config address types).  
7. **Privatize `pool.isPriceAboveNinetyNineCents`** after InvestPanel uses only `quoteTrade.priceAboveNinetyNineCents`.  
8. **Privatize `pool.estimatePriceAfterSwap`** after InvestPanel uses only `quoteTrade.priceAfter`.  
9. **Privatize `pool.outcomePrice` + `collateralReserve`** (internal to display/metrics).  
10. **Demote `listFactoryMarkets`** (document as advanced / plan FE migrate off create+fetch-market) so **`listMarkets` is the only catalog**.

Honorable mentions: privatize `withAddresses`, `submitCalls` visibility, `VoizMarket.outcome` / `VoizOutcome`; rename `Ponder*` DTOs; fold `aggregateOutcomeCollateral` into `MarketSummary`; drop `resolvedDisplayRows` once FE stops re-exporting.

---

## Out of scope / non-goals

- Do not delete implementations required by class methods (only visibility / barrel).  
- Smoke scripts may deep-import `dist/*`; that is not a reason to keep root exports.  
- Intentional encode+send pairs inside classes stay; just do not re-export Encode* param types.  
- No FE/SDK code changes in this audit pass.
