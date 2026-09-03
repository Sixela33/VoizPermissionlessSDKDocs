# Audit: less code — dead / redundant SDK surface

**Scope:** read-only. `sdkDev/sdk` (+ FE import evidence).  
**Date:** 2026-09-02 (America/Buenos_Aires)  
**Bias:** having less code is much better than having code. Prefer **delete** over demote; demote only when a live call site remains.  
**Do not edit SDK here** — checklist only.

Related (API/export-focused, not line-delete): `_audit-fe-dead-exports.md`, `_audit-duplicate-apis.md`.

---

## Snapshot

| Metric | Value |
|--------|------:|
| `src/**` (excl. tests) | ~5.9k lines |
| Root `*.mjs` + `.bak` + root unit test | ~966 lines |
| Local `dist/` (gitignored) | ~972K / 188 files |
| Heaviest dist dirs | uniswap ~180K, abis ~164K, market ~144K, actions ~124K, ponder ~120K |
| Public `src/index.ts` symbols unused by FE | **26 / ~42** (types-heavy) |
| FE deep import of SDK internals | **1** (`frontend/lib/market-summary-stats.test.ts` → `sdk/.../ponder/stats.ts`) |
| Smoke deep-imports of `dist/*` | `actions/seedMarket`, `actions/createMarket`, `abis/uniswap`, `uniswap/seed-plan` |

FE product path is already class-centric (`client` / `market` / `pool` / `user` + a handful of math/display helpers). Most of the waste is **orphaned encode paths, a dead outcome facade, legacy share-book, and one-off smoke scripts**.

---

## Top 10 deletions / collapses

### 1. Delete root smoke / quote / check scripts (+ `.bak`) — **~800–900 lines, ~10 files**

**Delete:**

- `smoke-gateoff-redeploy.mjs` (+ `.bak`)
- `smoke-continue.mjs`, `smoke-finish.mjs`
- `quote-smoke.mjs`, `quote-decode.mjs`, `quote-probe.mjs`
- `scripts-check-liq.mjs`, `check-pools.mjs`
- Optional: `price-after-swap.test.ts` (91 lines; deep-imports `./dist/uniswap/price-after-swap.js` — either move under `src/` with package test script or drop if covered by FE/manual)

**Why:** Not in `package.json` scripts. Hardcoded tokens/RPC/tmp JSON. Only things still deep-importing non-exported `dist/actions/*` and `dist/abis/*`. They **force** keeping dead encode APIs alive “for tooling.”

**Win:** ~15% of repo text outside `node_modules`; removes pressure to keep smoke-only exports.

**Risk:** Lose manual Base Sepolia regression recipes — park one short note in README or a private gist if needed.

---

### 2. Delete dead `addLiquidityToOutcomeCalls` (+ `AddLiquidityParams`) — **~70–90 lines**

**Evidence:** `addLiquidityToOutcomeCalls` has **zero callers** in `src`, FE, or smoke. Client no longer exposes `addLiquidityToOutcome`. Product LP is `VoizMarket.addLiquidityAcrossOutcomes` → `provideLiquidityCalls`.

**Delete:**

- `src/actions/liquidity.ts` lines ~14–91 (`addLiquidityToOutcomeCalls`)
- `AddLiquidityParams` in `src/types.ts`

**Keep:** `provideLiquidityCalls` (used by market across-outcomes).

**Win:** Removes a near-duplicate of provide+split that only exists as a parallel product path.

**Risk:** None for current FE. Advanced single-outcome LP would need to be rebuilt from `splitPositionCalls` + `provideLiquidityCalls` if ever wanted.

---

### 3. Delete `VoizOutcome` facade + `market.outcome()` — **~35–40 lines + README row**

**Evidence:** `VoizOutcome` only constructed in `VoizMarket.outcome()`. **Zero** FE/smoke callers of `.outcome(`. FE uses `client.pool(outcomeToken)` directly.

**Delete:**

- `src/market/outcome.ts` (entire file, 18 lines)
- `VoizMarket.outcome()` (~20 lines)
- Re-export in `src/market/index.ts`
- README “VoizOutcome” table row

**Win:** Removes a facade that adds an indirection with no behavior (`get pool()` → `new VoizPool`).

**Risk:** None today. Anyone wanting typed outcome handles can use `{ index, token, label }` + `client.pool(token)`.

---

### 4. Delete legacy trade-lot share book — **~190 lines impl + shrink/delete ~100 lines test**

**In `src/user/share-positions.ts`:**

- `applyTradeLots`, `buildUserSharePositions`, `ShareLot` (~lines 49–234 region)
- Comment already says prefer `buildUserSharePositionsFromIndexed`

**Keep:** `buildUserSharePositionsFromIndexed`, `normalizeMarketsFilter`, `avgEntryPriceCt` (still used by indexed path), types used by `VoizUser.positions`.

**Tests:** `share-positions.test.ts` mostly exercises the legacy path — rewrite to indexed helpers or delete.

**Win:** ~250–300 lines; one mental model for positions (indexer).

**Risk:** Offline/tooling that rebuilt lots from trades only — none in-repo.

---

### 5. Delete unused one-call / alias helpers — **~40–50 lines**

| Symbol | File | Callers |
|--------|------|---------|
| `splitPositionCall` | `actions/seedMarket.ts` | **none** (keep `splitPositionCalls`) |
| `redeemPositionsCall` | `actions/redeem.ts` | **none** (keep `redeemCalls`) |
| `encodeApproveCall` | `actions/approvals.ts` | only self inside `encodeApprovalCalls` — inline `encodeApprove(..., erc20Abi)` |
| `defaultOddsBps` | `uniswap/seed-plan.ts` | **none** (FE has `defaultOddsPct`) |

**Win:** Fewer parallel “with/without approvals” twins; smoke no longer excuses them.

---

### 6. Delete unused internal barrels — **~64 lines, 2 files**

**Nobody imports:**

- `src/uniswap/index.ts` (47) — all consumers import leaf modules
- `src/abis/index.ts` (17) — same; public API only re-exports `erc20Abi` from `./abis/erc20.js`

**Keep:** `market/index.ts`, `user/index.ts`, `ponder/index.ts`, `react/index.ts` (used by root / react entry).

**Win:** Stops implying a second public surface; slightly less dist noise.

---

### 7. Collapse encode + approvals into one module — **−1 file, ~10–20 net lines**

Today:

- `src/encode/calls.ts` — `encodeCall`, `encodeApprove`, `exactApprovalPlan`, `encodeExactApprovals`
- `src/actions/approvals.ts` — thin ERC20-specialized wrapper around the above

**Collapse:** move the four encode helpers into `actions/approvals.ts` (or `encode/approvals.ts`) and delete the empty `encode/` dir. All action modules already import approvals/encode together.

**Not worth collapsing (yet):** full `actions/*` into `market`/`client` — those encode batches are still the real implementation behind class methods.

**Win:** One less layer in the “encode → action → class” stack for the approvals slice.

---

### 8. Drop always-null `marketImageUrl` / empty `BLOB_PUBLIC_BASE_URL` — **~20 lines + simplify call sites**

`BLOB_PUBLIC_BASE_URL = ""` ⇒ `marketImageUrl` **always returns null**. FE already overlays blob URLs in `lib/fetch-market.ts` / `lib/market-image-url.ts` via `NEXT_PUBLIC_BLOB_PUBLIC_BASE_URL`.

**Delete / simplify:**

- `BLOB_PUBLIC_BASE_URL`, `marketImageUrl` in `ponder/market-image.ts`
- Call sites in `VoizMarket` summary RPC mapping that set `imageUri: marketImageUrl(...)` (always null)

**Keep public:** `marketImagePath` (FE + API route use it).

**Win:** Stops shipping a dead URL builder that duplicates FE env wiring.

---

### 9. Slash unused **public type** re-exports (barrel diet) — **~30–40 lines in `index.ts`, clearer API**

FE never imports these (26 symbols). Keep them **internal** to modules; do not delete the underlying types if class methods need them:

`CreateMarketParams/Args`, `SeedLeg/Plan`, `SeedMarket/OutcomeParams`, `V4PoolKey`, `ChainAddresses`, `AddressOverrides`, `ClientConfig`, `RedeemParams`, `LpPositionInput`, `ListMarketsResult`, `PoolStatus/State`, `QuoteExactInResult`, `OutcomePrice/MetricsRow`, `DisplayOutcome*`, `EncodeResolve/MergeParams`, `User*Params`, `MarketOutcomeStats`.

**Also:** `EncodeResolveParams` / `EncodeMergeParams` are already **private method** params on `VoizMarket` — exporting them from root is pure noise.

**Win:** Smaller public contract; pairs with prior dead-exports audit. Low line delete in implementations, high clarity.

---

### 10. Close the FE deep-import / API-gap — **move or delete 1 FE test; stop proving internals**

`frontend/lib/market-summary-stats.test.ts` imports:

```ts
from "../../sdkDev/sdk/src/ponder/stats.ts"  // buildMarketOutcomeStats, selectNewestPriceByOutcome
```

while `aggregateOutcomeCollateral` comes from the public barrel.

**Options (pick one, prefer less code):**

1. **Delete the FE test** and trust `MarketSummary` / card behavior (least code), or  
2. Move the test into `sdk/src/ponder/stats.test.ts` and keep helpers **unexported**.

Do **not** re-export `buildMarketOutcomeStats` / `selectNewestPriceByOutcome` on the public barrel just to satisfy the FE test (that was already demoted once).

**Win:** Removes the only FE→`sdk/src/**` deep path; kills the “API gap” excuse for publishing ponder internals.

---

## Extra collapse notes (not top-10, still real)

### Encode / action / class stack (keep implementations, stop treating as three APIs)

Live ownership today (post-trim client):

| Concern | Implementation | App entry |
|---------|----------------|-----------|
| Create | `createMarketCalls` | `client.createMarket` |
| Seed | `seedMarketCalls` / `seedOutcomeCalls` | `market.seed` |
| LP | `provideLiquidityCalls` | `market.addLiquidityAcrossOutcomes` |
| Redeem | `redeemCalls` | `client.redeem` |
| Burn | `encodeBurnPosition` | `client.burn` |
| Swap | `encodeSwapExactIn` | `pool.swap` / `quoteTrade` |

**Do not** delete `seedOutcomeCalls` — used by `seedMarketCalls` (and only smoke deep-imported it). After smoke deletion it stays internal-only.

**Optional later:** inline `lpMintAmounts` (9 lines) into `liquidity.ts` / market LP encode — tiny win.

### `uniswap/pool.ts` vs `market/pool.ts`

Not duplicates: low-level key/slot0 helpers vs `VoizPool` class. **Keep both.**

### Ponder weight (`src/ponder/client.ts` ~704 lines)

Heavy but **live** (listMarkets, summary, prices, user positions/trades). Not a delete candidate without rewriting the indexer client. Dist still emits full `dist/ponder/**` (~24 files / ~47–120K) though `./ponder` is not a package export — acceptable as internal build output; don’t re-add a public subpath.

### FE quote redundancy (InvestPanel)

`quoteTrade` already returns `priceAfter` + `priceAboveNinetyNineCents`; panel falls back to `estimatePriceAfterSwap` / `isPriceAboveNinetyNineCents`. **FE cleanup**, not SDK delete — pool helpers still used as fallbacks.

---

## Estimated win if top 10 landed

| Bucket | Files | Lines (vibe) |
|--------|------:|-------------:|
| Smoke/scripts/bak (+ optional root test) | 9–10 | ~800–950 |
| Dead LP path + types | 1–2 | ~80 |
| VoizOutcome | 1 + edits | ~40 |
| Legacy share book + test | 1–2 | ~250–300 |
| Dead aliases / `defaultOddsBps` | edits | ~50 |
| Unused barrels | 2 | ~64 |
| Encode∪approvals collapse | −1 dir | ~15 net |
| marketImageUrl null path | edits | ~20 |
| Public type barrel diet | `index.ts` | ~30–40 |
| FE deep-import test fix | 1 FE file | ~40–80 |
| **Total** | **~15–20** | **~1.3k–1.6k** |

Against ~5.9k `src` + ~0.9k root scripts ≈ **~20–25% less SDK tree text**, with almost no FE product-path churn (except optional test move/delete).

---

## Suggested delete order (safest → juiciest)

1. Smoke/scripts/bak (no import graph into `src`)  
2. `VoizOutcome` + `market.outcome`  
3. `addLiquidityToOutcomeCalls` + `AddLiquidityParams`  
4. Dead aliases (`splitPositionCall`, `redeemPositionsCall`, `defaultOddsBps`, inline `encodeApproveCall`)  
5. Unused `uniswap/index` + `abis/index`  
6. Legacy share-book + test rewrite  
7. `marketImageUrl` null path  
8. Encode/approvals collapse  
9. Public type barrel diet  
10. FE stats deep-import cleanup  

---

## Evidence roots

- SDK: `/home/s1x3l4/Desktop/voiz/permissionlessMarkets/sdkDev/sdk`
- FE: `/home/s1x3l4/Desktop/voiz/permissionlessMarkets/frontend`
- Public barrels: `src/index.ts`, `src/react/index.ts`, `package.json` `exports` (`.` + `./react` only)
- Grep anchors: `VoizOutcome`, `addLiquidityToOutcomeCalls`, `redeemPositionsCall`, `splitPositionCall`, `defaultOddsBps`, `from "../../sdkDev/sdk/src/ponder/stats.ts"`
