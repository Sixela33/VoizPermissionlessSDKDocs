# Duplicate / overlapping public API audit — `@voiz/markets-sdk`

**Scope:** read-only review of `sdk/src` public surface (`index.ts` + class methods).  
**Date:** 2026-09-02  
**Constraint:** no SDK source edits (parallel trim agent owns that).

## Ownership model (intended)

| Layer | Owns |
|-------|------|
| **VoizMarketsClient** | Addresses, `send`, factory create, catalog (`listMarkets`), cross-cutting encode wrappers, single-outcome / provide-liquidity primitives, redeem / burn |
| **VoizMarket** | Market-scoped load/summary/display, resolve/merge, **seed thin-wrap**, **primary LP** (`addLiquidityAcrossOutcomes`), RPC price rows |
| **VoizPool** | Live pool state, **quoteTrade**, swap encode/send, mid/after-swap helpers |
| **VoizUser** | Indexed trades / share book / LP discovery |
| **Standalones (`*Calls`, uniswap/*, ponder/*)** | Implementation + AA/tooling; not the default app path |

---

## Groups

### 1. Seed — `client.seedMarket` vs `market.seed` — **KEEP BOTH (intentional)**

| Symbol | Location | Role |
|--------|----------|------|
| `VoizMarketsClient.seedMarket` / `encodeSeedMarket` | `client.ts` | Send + encode; used by FE create + LiquidityPanel |
| `VoizMarket.seed` | `market.ts` | Thin wrap: injects `this.address` → `client.seedMarket` |
| `seedMarketCalls` / `seedOutcomeCalls` / `splitPositionCalls` | `actions/seedMarket.ts` | Root-exported standalones |
| `splitPositionCall` | same | Approve-free single call; subset of `splitPositionCalls` |

**Canonical (product):** `client.seedMarket` (and `market.seed` when you already hold a `VoizMarket`).  
**Keep both client + market** as intentional dual entry.  
**Thin-wrap / un-export candidates:** root `splitPositionCall` (keep `splitPositionCalls`); optionally stop root-exporting `seedOutcomeCalls` if only used by `seedMarketCalls` + scripts.  
**Risk if removed:** `market.seed` — low (FE uses client today); `seedOutcomeCalls` — medium for scripts (`scripts-check-liq.mjs`).

---

### 2. Encode X on client vs market vs standalone

Pattern: **standalone `*Calls` → client.`encode*` → (sometimes) market.`encode*` / send**.

| Concern | Standalone (root) | Client | Market |
|---------|-------------------|--------|--------|
| Create | `createMarketCalls` | `encodeCreateMarket` / `createMarket` | — |
| Seed | `seedMarketCalls` | `encodeSeedMarket` / `seedMarket` | `seed` (send only) |
| Split | `splitPositionCalls` | `encodeSplitPosition` | (used inside LP encode) |
| Single-outcome LP | `addLiquidityToOutcomeCalls` | `encodeAddLiquidityToOutcome` / `addLiquidityToOutcome` | — |
| Provide LP (no split) | `provideLiquidityCalls` | `encodeProvideLiquidity` | used by across-outcomes |
| Across-outcomes LP | — | — | `encodeAddLiquidityAcrossOutcomes` / `addLiquidityAcrossOutcomes` |
| Redeem | `redeemCalls` / `redeemPositionsCall` | `encodeRedeem` / `redeem` | — |
| Burn | `encodeBurnPosition` | `encodeBurn` / `burn` | — |
| Swap | `encodeSwapExactIn` | — | (via) `pool.encodeSwap` / `swap` |
| Resolve / merge | — | — | `encodeResolve` / `encodeMerge` + send |

**Canonical:** class methods for apps; standalones for AA/custom senders.  
**Action:** treat client `encode*` as thin wrappers (keep). Prefer **un-exporting** low-level duplicates from root when a class wrapper exists and FE does not import them: `encodeBurnPosition` (use `client.encodeBurn`), `encodeSwapExactIn` (use `pool.encodeSwap`), `redeemPositionsCall` (use `redeemCalls` / `client.encodeRedeem`), `splitPositionCall`.  
**Risk:** low for FE (no direct imports found); medium for external AA integrators / smoke scripts.

---

### 3. LP naming cluster (`addLiquidity*` / `provideLiquidity*`)

| Symbol | Intended use |
|--------|----------------|
| **`market.addLiquidityAcrossOutcomes`** (+ encode) | **Primary product LP** (README + comments) — FE LiquidityPanel |
| `client.addLiquidityToOutcome` (+ encode + `addLiquidityToOutcomeCalls`) | Advanced single-outcome split+mint |
| `client.encodeProvideLiquidity` / `provideLiquidityCalls` | Building block (permit2 + mint, no split) |

**Canonical:** `VoizMarket.addLiquidityAcrossOutcomes`.  
**Un-export / demote:** keep single-outcome + provideLiquidity **implemented**, but consider **removing from root public export** (`addLiquidityToOutcomeCalls`, `provideLiquidityCalls`) and documenting as advanced client methods only — or rename later for clarity (`mintLiquidityIntoPool` vs `addLiquidityToMarket`).  
**Risk if removed from root:** low for current FE; client methods still needed by market across-outcomes encode.  
**Do not remove** client `encodeProvideLiquidity` without relocating call site inside market/actions.

---

### 4. Quote / price paths

| Symbol | Role |
|--------|------|
| **`pool.quoteTrade`** | Canonical exact-in quote (+ range cap + spot/priceAfter enrichment) — FE InvestPanel |
| `pool.encodeSwap` / `swap` | Canonical trade send |
| `pool.estimatePriceAfterSwap` | Standalone wrap; **overlaps** `quoteTrade`’s `priceAfter` |
| `pool.isPriceAboveNinetyNineCents` | Also on quote result as `priceAboveNinetyNineCents` |
| `estimateOutcomePriceAfterSwap` / `isOutcomePriceAboveNinetyNineCents` / `encodeSwapExactIn` | Root standalones used by pool |
| `market.displayPrices` | Ponder→RPC display — FE market page / InvestPanel |
| `market.outcomePrices` | RPC-only; fallback inside `displayPrices` |
| `pool.outcomePrice` / `priceFromState` | Per-pool mid |

**Canonical:** `quoteTrade` for trading; `displayPrices` for UI mids.  
**Thin-wrap:** keep pool helpers as convenience; **un-export** raw `estimateOutcomePriceAfterSwap`, `isOutcomePriceAboveNinetyNineCents`, `encodeSwapExactIn` from root (or move behind `/uniswap` subpath only — not currently separate).  
**Optional:** mark `outcomePrices` as internal / stop advertising; keep as `displayPrices` fallback.  
**Risk:** low — FE does not import the standalones.

---

### 5. User positions / trades vs client.user vs ponder / builders

| Symbol | Role |
|--------|------|
| **`client.user(addr).trades` / `positions` / `lpPositions`** | Canonical app API — holdings, profile, LiquidityPanel |
| `fetchTradesFromPonder` / `fetchSharePositionsFromPonder` / `fetchUserPositionsFromPonder` / `fetchAllUserPositionsFromPonder` / `fetchMarketOutcomeContextFromPonder` | Raw indexer — wrapped by VoizUser |
| `market.listLpPositions` | Enrich-only (caller supplies tokenIds); used by `user.lpPositions` |
| `enrichLpPosition` | Lower-level enrich |
| `buildUserSharePositionsFromIndexed` | Used by `user.positions` |
| `buildUserSharePositions` + `applyTradeLots` | **Legacy trade-lot path** — tests only; comment says prefer indexed |

**Canonical:** `VoizUser.*`.  
**Un-export from root:** ponder fetch helpers that User already wraps (keep available via `@voiz/markets-sdk/ponder` if needed); `buildUserSharePositions`, `applyTradeLots` (keep in module for tests, drop from main export); optionally `enrichLpPosition` if `listLpPositions` stays.  
**Keep public:** `market.listLpPositions` only if external indexers feed positions without User — else thin-wrap / package-private.  
**Risk:** medium if any external consumer used raw ponder from main entry; **low for current FE** (uses `client.user`). Removing legacy builders breaks `share-positions.test.ts` unless tests import deep paths.

---

### 6. Catalog: `listMarkets` vs ponder / RPC siblings

| Symbol | Role |
|--------|------|
| **`client.listMarkets`** | Ponder-first + RPC fallback — FE `useMarkets` |
| `fetchMarketSummariesFromPonder` | Raw; wrapped by listMarkets |
| `client.listMarketSummariesFromRpc` / `listFactoryMarkets` / `isMarket` | RPC path / factory gate |
| `market.summary` | Ponder-first detail |
| `market.summaryFromRpc` | RPC detail — FE `fetch-market.ts` still uses this for blob URL overlay |
| `fetchMarketByIdFromPonder` | Raw detail |
| `market.priceHistory` | Soft-fail ponder wrap of `fetchPriceHistoryFromPonder` |

**Canonical:** `listMarkets` + `market.summary` / `priceHistory`.  
**Un-export from main:** `fetchMarketSummariesFromPonder`, `fetchMarketByIdFromPonder`, `fetchPriceHistoryFromPonder`, `fetchLatestOutcomePricesFromPonder`, `fetchOutcomeMetricsFromPonder` (already wrapped by market display*). Prefer `@voiz/markets-sdk/ponder` for power users.  
**Risk:** low for FE list path; keep `summaryFromRpc` until FE blob overlay moves into SDK or FE uses `summary()` + local image overlay only.

---

### 7. Approvals: `encodeApprove` vs `encodeApproveCall` vs `encodeApprovalCalls`

| Symbol | Role |
|--------|------|
| `encodeApprove` | Generic ABI approve (`encode/calls.ts`) |
| `encodeApproveCall` | Same + hardcodes `erc20Abi` |
| `encodeApprovalCalls` | Force / read-allowance planner |
| `encodeExactApprovals` / `exactApprovalPlan` | Pure plan |

**Canonical for apps:** `encodeApprovalCalls` (or none — prefer class flows).  
**Un-export:** `encodeApproveCall` (alias of `encodeApprove`+erc20) **or** `encodeApprove` from root; keep one. Keep `exactApprovalPlan` only if external AA needs it.  
**Risk:** low.

---

### 8. Pool reads vs uniswap pool helpers

| Class | Standalone |
|-------|------------|
| `pool.state()` | `getSlot0`, `getLiquidity`, `isPoolInitialized` |
| `pool.key` | `poolKey`, `poolId`, `sortTokens` |

**Canonical:** `VoizPool.state()`.  
**Un-export:** `getSlot0` / `getLiquidity` / `isPoolInitialized` from main (still used internally). Keep `poolKey` / math helpers if scripts need them.  
**Risk:** low for FE; medium for tooling that reads slot0 directly.

---

### 9. `planSeed` vs `planMarketSeed`

| `client.planSeed` | Injects collateral from addresses |
| `planMarketSeed` | Standalone (scripts) |
| `previewSeedTotals` / `normalizeOddsBps` | FE SeedFields / create |

**Canonical math:** `planMarketSeed` / preview helpers.  
**Thin-wrap:** keep `client.planSeed` or drop as redundant with `encodeSeedMarket`’s plan return.  
**Risk:** low.

---

## Top recommendations (actionable)

1. **Freeze ownership in docs / export policy:** apps → `client` / `market` / `pool` / `user` only; raw `fetch*FromPonder` and most `*Calls` live on `@voiz/markets-sdk/ponder` or stay unexported from `.`.
2. **LP:** document one product name — `market.addLiquidityAcrossOutcomes`. Demote `addLiquidityToOutcome*` + `provideLiquidityCalls` to advanced (client-only / no root export). Do not delete implementations.
3. **Ponder:** stop main-entry export of fetches already wrapped (`listMarkets`, `market.summary` / `displayPrices` / `displayMetrics` / `priceHistory`, `user.trades|positions|lpPositions`). Use `./ponder` subpath for escape hatch.
4. **Legacy share book:** un-export `buildUserSharePositions` + `applyTradeLots` from main (tests import deep or stay in package private); keep `buildUserSharePositionsFromIndexed` internal to User unless needed.
5. **Approvals / swap / burn standalones:** un-export `encodeApproveCall` (or `encodeApprove`), `encodeSwapExactIn`, `encodeBurnPosition`, `splitPositionCall`, `redeemPositionsCall` — class wrappers remain.
6. **Quote:** treat `pool.quoteTrade` as sole trade quote API; un-export `estimateOutcomePriceAfterSwap` / `isOutcomePriceAboveNinetyNineCents`; optionally soft-deprecate standalone `pool.estimatePriceAfterSwap` in docs (still fine as convenience).
7. **KEEP BOTH** `client.seedMarket` and `market.seed` (intentional). Prefer one in FE for consistency (`client.seedMarket` today).
8. **Do not remove** `client.encodeProvideLiquidity` / `encodeSplitPosition` without inlining — market across-outcomes depends on them.
9. **`outcomePrices` vs `displayPrices`:** keep both internally; advertise only `displayPrices` (+ `quoteTrade` for trades).
10. **`market.listLpPositions`:** keep as enrich primitive for User; avoid documenting as parallel to `user.lpPositions`.

### Explicit non-goals / safe leaves

- Intentional dual seed entry (client + market).
- Encode + send pairs on the same owner (`encodeX` + `X`) — not duplication of ownership, AA pattern.
- Uniswap tick/liquidity math exports (real library surface).
- ABI / chain address exports.

### FE blast radius (current)

Uses class APIs almost exclusively (`listMarkets`, `seedMarket`, `addLiquidityAcrossOutcomes`, `quoteTrade`, `displayPrices`, `user.*`, `burn`, `merge`). Root standalones still imported: `normalizeOddsBps`, `previewSeedTotals`, `aggregateOutcomeCollateral`, `buildMarketOutcomeStats`, `selectNewestPriceByOutcome`, `resolvedDisplayRows`, `marketImagePath`, types/abis. Trimming ponder/`*Calls`/legacy builders from main export is **low FE risk** if those named helpers stay.
