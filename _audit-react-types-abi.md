# React / types / ABI / chains public-surface audit — `@voiz/markets-sdk`

**Scope:** read-only review of React barrel (`./react`), exported types, `chains.ts` / addresses, and ABI public surface vs FE usage.  
**Date:** 2026-09-02  
**Paths:** SDK `sdkDev/sdk`, FE `permissionlessMarkets/frontend`  
**Constraint:** no SDK source edits in this pass.

Companion: `_audit-duplicate-apis.md` (action / class overlap). This doc does **not** re-litigate `*Calls` vs class methods except where barrels leak internals.

---

## Method

- Inventory `src/index.ts`, `src/react/*`, `src/types.ts`, `src/chains.ts`, `src/abis/*`.
- Scan FE `app/`, `components/`, `lib/` for `from "@voiz/markets-sdk"` and `from "@voiz/markets-sdk/react"` (no `/ponder` imports).
- Cross-check root exports never imported by FE — treat as **candidates**, not auto-delete (scripts / external integrators may still need them).

### FE React imports (complete)

| Symbol | Files |
|--------|--------|
| `VoizProvider` | `VoizProviderBridge.tsx` |
| `VoizSendCalls` | `VoizProviderBridge.tsx` |
| `useVoiz` | hooks + create/profile/markets pages + Invest/Liquidity/Resolve/PriceHistory |

**Not imported by FE:** `useVoizClient`, `VoizContextValue`, `VoizProviderProps`, `SendCallsOptions` (from `/react`).

### FE root imports touching this audit

`VoizMarketsClient`, `Call`, `ResolverKind`, `MarketSummary`, `UserSharePosition`, `AllMarketsLpPosition`, `LpPosition`, `PonderPricePoint`, `PonderTrade`, `getAddresses`, `erc20Abi`, plus a few math/ponder helpers (`normalizeOddsBps`, `previewSeedTotals`, `aggregateOutcomeCollateral`, `buildMarketOutcomeStats`, `selectNewestPriceByOutcome`, `marketImagePath`, `resolvedDisplayRows` via FE re-export).

**ABI from SDK:** only `erc20Abi`.

---

## 1. React exports

| Symbol | Verdict | Notes |
|--------|---------|--------|
| `VoizProvider` | **KEEP** | Sole FE injection point (`VoizProviderBridge`). |
| `useVoiz` | **KEEP** | Canonical FE hook. Almost always `const { client } = useVoiz()`; Invest/Liquidity also take `addresses`. |
| `useVoizClient` | **REMOVE** (or deprecate → remove) | Pure `useVoiz().client`. **0 FE call sites.** README still advertises it as peer of `useVoiz`. Redundant public API. |
| `VoizProviderProps` | **KEEP** | Useful for app wrappers; FE does not import today. |
| `VoizContextValue` | **KEEP** (type) / **THIN docs** | FE never imports the type. Runtime value exposes `client`, `addresses`, `chainId`, `sendCalls?`, `publicClient`, `walletClient?`. FE only needs `client` (+ occasional `addresses`). |
| `VoizSendCalls` | **KEEP** (short-term) / **RENAME later** | Alias of root `SendCalls`. FE bridge types against `VoizSendCalls`. Prefer converging on `SendCalls` and dropping the alias in a breaking pass. |
| `SendCallsOptions` (re-export from `/react`) | **KEEP** | Needed next to `VoizSendCalls` / provider `sendCalls`. Duplicate of root export — fine for ergonomics. |

### Context shape cleanup (actionable)

| Field on `VoizContextValue` | Verdict | Why |
|-----------------------------|---------|-----|
| `client` | **KEEP** | Product path. |
| `addresses` | **KEEP** | Used by Invest/Liquidity (`sdkAddresses.collateral`). Prefer documenting `client.addresses` as equivalent. |
| `chainId` | **KEEP** | Cheap; unused by FE today but valid. |
| `sendCalls` | **REMOVE from context value** (breaking) or mark internal | Comment already says prefer `client.send`. FE never reads it from context (injects via provider props only). |
| `publicClient` / `walletClient` | **REMOVE from context value** (optional breaking) | FE has its own `lib/public-client` and Privy wallet path. Duplicates `client.publicClient` / `client.walletClient`. |

**Canonical React story (proposed):**  
`<VoizProvider>` + `useVoiz().client` only. Drop `useVoizClient` from README + barrel.

---

## 2. Exported types

### Leftovers / aliases

| Symbol | Verdict | Notes |
|--------|---------|--------|
| `OutcomePositionMeta` | **REMOVE** from public barrel | Already `@deprecated` alias of `PonderOutcomeContext` (`share-positions.ts`). **0 FE imports.** Keep as internal alias only if call sites inside `user/` still want the name; stop re-exporting from `user/index` + root `index.ts`. |
| `PonderOutcomeContext` | **KEEP** | Canonical name for share-position enrichment. |
| `VoizSendCalls` | see React | Duplicate of `SendCalls`. |

### Confusing / overlapping names

| Pair | Verdict | Notes |
|------|---------|--------|
| `MarketSnapshot` vs `MarketSummary` | **KEEP both** + **JSDoc clarify** | Snapshot = on-chain `VoizMarket.load()` cache. Summary = catalog DTO (ponder/RPC) used everywhere in FE. Names are easy to swap mentally — strengthen one-line JSDoc on both. |
| `OutcomePriceRow` / `OutcomeMetricsRow` vs `DisplayOutcomePrices` / `DisplayOutcomeMetrics` | **KEEP** | FE does not import; used by `VoizMarket` display APIs. Optional rename later: `MarketOutcomePriceRow` if docs stay confusing. |
| `UserSharePosition` vs `PonderSharePositionRow` vs `OutcomeBalance` | **KEEP** / audit `OutcomeBalance` | `UserSharePosition` used by FE holdings. `OutcomeBalance` appears unused by FE — **candidates for un-export** if only internal to market balance helpers. |
| `SeedLeg` / `SeedPlan` / `V4PoolKey` | **KEEP** | Single definitions in `types.ts`; uniswap modules re-export types (not duplicate defs). Root exports once from `types.js` — good. |
| `CreateMarketParams` vs `CreateMarketArgs` | **KEEP Params** / **demote Args** | `Args` is factory calldata shape. FE never imports. Consider stopping root export (advanced encode only). |
| `AddLiquidityParams` | **KEEP** (client) / clarify JSDoc | Comment already says prefer across-outcomes. Name still reads as “the” LP API — see duplicate-apis audit. |
| `ListMarketsResult` | **KEEP** | Return type of `listMarkets`; FE uses method, may not import type. |
| `ClientConfig` / `AddressOverrides` / `SendCalls` / `SendCallsOptions` | **KEEP** | Constructor / sender surface. |
| `UserPositionsParams` / `UserTradesParams` / `UserLpPositionsParams` | **KEEP** | Used with `VoizUser`; FE may only see inferred types. |
| `ResolverKind` | **KEEP** | FE create page. |
| `Call` | **KEEP** | FE bridge. |

### Types FE never imports (safe demote candidates)

Not automatically dead — class methods still return/accept them — but **do not need root export** if apps only touch classes:

- Encode param bags rarely imported at app layer: `SeedMarketParams`, `SeedOutcomeParams`, `RedeemParams`, `CreateMarketArgs`
- Pool internals: `PoolStatus`, `PoolState`, `QuoteExactInResult` (unless app types quotes explicitly)
- Display row types (unless apps annotate)

**Action:** prefer exporting **product DTOs** (`MarketSummary`, `UserSharePosition`, `LpPosition`, `AllMarketsLpPosition`, trade/price ponder types) from root; leave encode param types on class method signatures without root re-export where possible.

---

## 3. `chains.ts` / addresses

| Symbol / item | Verdict | Notes |
|---------------|---------|--------|
| `getAddresses` | **KEEP** | FE `lib/chain.ts`, `FaucetButton`. |
| `registerAddresses` | **KEEP** | 0 FE imports; needed after redeploy / scripts. |
| `DEFAULT_CHAIN_ID` | **KEEP or document** | Equals Base Sepolia. FE does **not** use it — FE `CHAIN_ID` defaults toward **Base mainnet** unless `NEXT_PUBLIC_CHAIN=sepolia`. Easy footgun if someone constructs `new VoizMarketsClient()` without `chainId`. |
| `base` / `baseSepolia` re-exports | **REMOVE** from root | FE imports from `viem/chains` directly. Re-exporting chain objects adds noise and version-skew risk. |
| `BASE_SEPOLIA` comment (`baseSepolia-clean/manifest.json`, “FreshStack”) | **UPDATE comment** | Path/name may be stale; keep “override via `getAddresses` / `registerAddresses`” guidance. |
| `BASE_MAINNET` comment (“not published in the frontend yet”) | **UPDATE comment + fill or gate** | **Stale:** FE already defaults to mainnet (`IS_BASE_MAINNET`). SDK still ships **zero** market addresses for `8453`. Either publish addresses or make `getAddresses(8453)` fail loudly until registered — zeros silently break create/seed. |
| `ChainAddresses.marketView` | **REMOVE** (or stop requiring) | Present in type + maps; **never read** by SDK `src/`. Dead public field. |
| `ChainAddresses.wrapped1155Factory` | **REMOVE** (or stop requiring) | Same — written in `chains.ts` / `types.ts` only. |
| Uniswap v4 periphery on both chains | **KEEP** | Real deployments; used. |
| `hook` / `seerV4Swap` zeros on mainnet | **KEEP shape** / fix values | Required for product; zero hook correctly fails via `assertHook` on mint paths. |

### FE address usage pattern

- Provider: `getAddresses(chainId)` defaults inside `VoizProvider` (bridge does not pass `addresses`).
- FE still keeps `lib/chain.ts` `addresses.collateral` via `getAddresses(CHAIN_ID)` for faucet/profile paths without a client.
- Invest/Liquidity: `useVoiz().addresses.collateral` (parallel to `client.addresses`).

**Action:** one documented path — prefer `client.addresses`; treat FE `lib/chain.addresses` as transitional.

---

## 4. ABI exports

Root barrel currently exports:

`erc20Abi`, `marketFactoryAbi`, `marketAbi`, `routerAbi`, `adminOracleAbi`, `multisigOracleAbi`, `conditionalTokensAbi`, `permit2Abi`, `positionManagerAbi`, `poolManagerAbi`, `stateViewAbi`, `v4QuoterAbi`, `seerV4SwapAbi`

| Symbol | FE import? | SDK internal? | Verdict |
|--------|------------|---------------|---------|
| `erc20Abi` | **yes** (faucet, transfer, profile, dildo balance) | yes | **KEEP** on root (or document `@voiz/markets-sdk` ABI exception). |
| All other `*Abi` | **no** | yes (actions/market/pool/client) | **REMOVE from root barrel** — move to `export … from "./abis"` only via new subpath **`@voiz/markets-sdk/abis`** *or* leave unexported and import privately inside SDK. |

### `abis/index.ts` extras not on root

| Symbol | Verdict |
|--------|---------|
| `createMarketParamsComponents` | **KEEP internal** — do not promote to root. |
| `FULL_RANGE_TICK_LOWER` / `FULL_RANGE_TICK_UPPER` | **KEEP internal** — prediction pools use prediction-range ticks; full-range constants are easy to misuse if public. |

**FE answer (Q4):** yes, FE imports ABI from SDK — **only `erc20Abi`**. No other ABI imports.

---

## 5. General cleanliness

### Confusing names (React / types / ABI slice)

| Issue | Action |
|-------|--------|
| `useVoiz` vs `useVoizClient` | Remove client shorthand. |
| `VoizSendCalls` vs `SendCalls` | One name (`SendCalls`). |
| `OutcomePositionMeta` | Delete public alias. |
| `MarketSnapshot` / `MarketSummary` | JSDoc disambiguation. |
| Root exporting every Uniswap/ponder helper | Out of scope here; see duplicate-apis audit — still makes types/ABI hard to see in `index.ts`. |

### Missing JSDoc on public methods (high-traffic)

Gaps that hurt the “use class methods” story:

| Location | Methods lacking real `/** */` |
|----------|-------------------------------|
| `VoizMarketsClient` | `withAddresses`, `encodeSeedMarket`, `encodeSplitPosition`, `encodeSeedOutcome`, `encodeRedeem`, `encodeBurn`, `planSeed`, `pool`, `market`, `user`, `redeem` (section `//` banners do not count) |
| `VoizPool` | `state`, getter `key` |
| `VoizUser` | overload surface is documented; OK overall |

`VoizMarket` public methods are generally well documented — **model to copy**.

### Barrel files exporting internals

| Barrel | Assessment |
|--------|------------|
| `src/react/index.ts` | **Clean** — 6 exports; only `useVoizClient` is surplus. |
| `src/abis/index.ts` | Fine as **internal** aggregator; should not all be re-exported from root. |
| `src/index.ts` | **Overloaded** — mixes product classes, ponder fetchers, uniswap math, raw ABIs, chain re-exports. For *this* audit: strip ABI (except maybe `erc20Abi`), strip `base`/`baseSepolia`, strip `OutcomePositionMeta`. |
| `src/user/index.ts` | Re-exports deprecated `OutcomePositionMeta` — stop. |
| `package.json` exports | `"."`, `"./react"`, `"./ponder"` — FE never uses `./ponder` (imports ponder symbols from root). Optional: stop re-exporting ponder from root and push apps to `./ponder` **or** keep root convenience and treat `./ponder` as advanced. |

### `assertHook`

Not public — **good**. Keep private.

---

## Priority action list

### P0 — clear wins (low FE risk)

1. **Remove** public `useVoizClient` (update README React section).
2. **Remove** public `OutcomePositionMeta` re-exports.
3. **Stop root-exporting** protocol ABIs except `erc20Abi` (or add `./abis` subpath and migrate FE only if needed — FE already only needs `erc20Abi`).
4. **Stop root-exporting** `base` / `baseSepolia`.
5. **Fix** Base mainnet comment + zero-address behavior (`getAddresses(8453)`).
6. **Drop or optionalize** dead `ChainAddresses` fields: `marketView`, `wrapped1155Factory`.

### P1 — clarify without large breaks

7. JSDoc pass on `VoizMarketsClient` encode/send entrypoints + `pool`/`market`/`user` factories.
8. Disambiguate `MarketSnapshot` vs `MarketSummary` in types.
9. Document canonical React usage: `useVoiz().client` only; context fields beyond `client`/`addresses` as advanced.
10. Converge `VoizSendCalls` → `SendCalls` (FE one-line type import change).

### P2 — optional thinning

11. Remove `sendCalls` / `publicClient` / `walletClient` from `VoizContextValue` (breaking for any external context consumers; FE OK).
12. Un-export rarely imported encode param types from root (`CreateMarketArgs`, seed/redeem param types) while keeping them as method params.
13. Decide root vs `./ponder` for indexer helpers (FE currently root-only).

---

## FE usage cheat-sheet (React)

```
VoizProviderBridge
  └─ <VoizProvider chainId publicClient sendCalls>
       └─ useVoiz() → { client [, addresses] }
            ├─ client.listMarkets / market / user / pool / createMarket / seedMarket / …
            └─ addresses.collateral (Invest/Liquidity only)
```

No `useVoizClient`. No SDK ABI except `erc20Abi`. No `@voiz/markets-sdk/ponder` imports.

---

## Out of scope (point elsewhere)

- Duplicate `seedMarket` / LP / quote APIs → `_audit-duplicate-apis.md`
- Whether to delete root `*Calls` / uniswap math exports → same
- Filling real Base mainnet Voiz addresses → deploy/manifest ownership, not rename work
