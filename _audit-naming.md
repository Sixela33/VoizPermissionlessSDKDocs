# Audit: naming / abstraction — `@voiz/markets-sdk`

**Scope:** read-only review of public + class surface names vs intended layers  
`Client → Market → Pool → User`, FE via `useVoiz().client`, hide Uniswap / encode / Ponder brands, `quoteX` ↔ write pairing.  
**Date:** 2026-09-02 (America/Buenos_Aires)  
**Constraint:** no SDK source edits in this pass — proposals only.  
**FE surveyed:** `permissionlessMarkets/frontend` (25 SDK importers). Smoke `*.mjs` noted separately.

---

## Intended naming rules (checklist)

| Rule | Pass if… |
|------|----------|
| Layer nouns | Methods live on the right object (`client` / `market` / `pool` / `user`) |
| No infra brands | Public names avoid `Ponder*`, `Uniswap*`, `Encode*`, `Factory*` (contract), `ExactIn` quoter jargon |
| quote ↔ write | Read preview `quoteX` pairs with mutating `X` / `writeX` on same layer |
| Product vocabulary | Prefer `trade`, `liquidity`, `seed`, `share`, `indexer` over implementation details |
| One canonical path | Duplicate RPC vs indexer helpers: one public name; the other private |

---

## Ranked rename table (top 12)

Priority = abstraction break severity × FE/product visibility (not raw call count).  
**Blast** = FE files/symbols that must change (smoke/scripts extra).

| # | Old | New (proposal) | Why | FE blast | Abstraction flag |
|---|-----|----------------|-----|----------|------------------|
| 1 | `PonderTrade` | `Trade` (or `UserTrade`) | Public type leaks indexer product brand | `lib/holdings.ts` (import + local type); README | **Breaks** — brand on barrel |
| 2 | `PonderPricePoint` | `PricePoint` (or `PriceHistoryPoint`) | Same brand leak; chart DTO | `lib/price-history.ts` (~7 refs); type-only | **Breaks** — brand on barrel |
| 3 | `source: "ponder" \| "rpc"` | `source: "indexer" \| "rpc"` | Union string leaks Ponder into every display/catalog result | `useMarkets.ts` default `"ponder"`; `markets/[address]/page.tsx` `data.source === "ponder"`; InvestPanel cadence if any | **Breaks** — brand in runtime values |
| 4 | `addLiquidityAcrossOutcomes` | `addLiquidity` | Primary LP write; name encodes algorithm (“across outcomes”) | `LiquidityPanel.tsx` (1 call) | **Breaks** — product verb buried under impl detail |
| 5 | `isPriceAboveNinetyNineCents` (+ field `priceAboveNinetyNineCents`) | `isBuyPriceCapped` / field `buyPriceCapped` *(or `isAboveMaxTradePrice`)* | Policy threshold hard-coded in identifier; Uniswap-pool helper dressed as product API | `InvestPanel.tsx` (method + `q.priceAboveNinetyNineCents`) | **Breaks** — domain rule in name; brittle if cap ≠ 0.99 |
| 6 | `user.positions` | `sharePositions` | Ambiguous vs LP; FE `marketKeys.positions` already means **LP** | `lib/holdings.ts` (1 call + locals) | **Breaks** — colliding “positions” vocab with `lpPositions` / FE keys |
| 7 | `listFactoryMarkets` | `listMarketAddresses` | “Factory” = on-chain contract role, not product noun; overlaps `listMarkets` | `lib/fetch-market.ts`, `create/page.tsx` (~5) | Mild brand / layer noise on Client |
| 8 | `estimatePriceAfterSwap` | Prefer fold into `quoteTrade` → use `priceAfter`; else rename `quotePriceAfter` | Orphan quote helper; duplicates `QuoteExactInResult.priceAfter` already filled by `quoteTrade` | `InvestPanel.tsx` (1; already prefers quote field) | Pairing gap: quote-ish name without `quote` prefix; redundant with `quoteTrade` |
| 9 | `EncodeResolveParams` / `EncodeMergeParams` | `ResolveParams` / `MergeParams` | `Encode*` names encoding layer; encode methods are already `private` | **0 FE** (types unused) | **Breaks** — encode brand on public barrel |
| 10 | `SeedMarketParams` | `SeedParams` | Redundant “Market”; `market.seed(Omit<SeedMarketParams,"marketAddress">)` reads awkward | **0 FE** direct (type export) | Mild; params should match `market.seed` |
| 11 | `outcomePrices` | demote private **or** rename `livePrices` | RPC-only twin of canonical `displayPrices`; FE already uses `displayPrices` only | **0 FE** | Dual public API; violates “one canonical path” |
| 12 | `QuoteExactInResult` | `QuoteTradeResult` | “ExactIn” = Uniswap Quoter jargon; pairs poorly with `quoteTrade`/`swap` | FE infers type (InvestPanel uses fields, no named import) — **low** | Infra jargon on public type |

### Honorable mentions (just below cut)

| Old | New | Note |
|-----|-----|------|
| `listLpPositions` (Market) | `enrichLpPositions` or package-private | Discovery is User/indexer; Market only enriches — name sounds like listing |
| `DisplayOutcomePrices` / `DisplayOutcomeMetrics` | `OutcomePrices` / `OutcomeMetrics` | “Display” is FE-centric; keep `source` field |
| `quoteSeed` | keep, but document as `quote` pair for `market.seed` | Pairing OK; naming asymmetry (`quoteSeed` vs `seed`) acceptable |
| `AllMarketsLpPosition` | `LpPositionAcrossMarkets` | Verbose but clear; low urgency |
| Internal `src/uniswap/*`, `src/ponder/*`, `src/encode/*` | keep folders | OK **internal**; do not re-export path brands on public API |
| Pool private `withDisplayPrices` | `withQuotePrices` | Misleading — enriches trade quote, not `market.displayPrices` |

---

## Layer map (current public verbs)

| Layer | Reads | Writes | Naming notes |
|-------|-------|--------|--------------|
| **Client** | `listMarkets`, `listFactoryMarkets`, `listMarketSummariesFromRpc`, `isMarket`, balances | `createMarket`, `redeem`, `burn` | Catalog OK; factory name noisy; encode\* mostly removed from class (good) |
| **Market** | `summary`, `priceHistory`, `displayPrices`, `displayMetrics`, `outcomePrices`, `seedStatus`, `canResolve`, `listLpPositions` | `seed`, `resolve`, `merge`, `addLiquidityAcrossOutcomes` | Display\* good product names; LP write too long; outcomePrices redundant |
| **Pool** | `state`, `outcomePrice`, `collateralReserve`, `quoteTrade`, `estimatePriceAfterSwap`, `isPriceAboveNinetyNineCents` | `swap` | `quoteTrade`↔`swap` is the gold-standard pair |
| **User** | `trades`, `positions`, `lpPositions` | — | Share vs LP naming collision |

---

## Brand / abstraction break flags (detail)

### Ponder\*

- **Exported types:** `PonderTrade`, `PonderPricePoint` (barrel).  
- **Runtime unions:** `source: "ponder"` on `ListMarketsResult`, `DisplayOutcomePrices`, `DisplayOutcomeMetrics`.  
- **Internal OK:** `fetch*FromPonder`, `isPonderConfigured`, module path `src/ponder/` — keep hidden.  
- **FE already coupled** to `"ponder"` string for refetch gating (`markets/[address]/page.tsx`).

### Encode\*

- Public types `EncodeResolveParams`, `EncodeMergeParams` while `encodeResolve` / `encodeMerge` / `encodeSwap` / `encodeAddLiquidityAcrossOutcomes` are **private** — type prefix is stale branding.  
- Folder `src/encode/calls.ts` + helper `encodeCall` fine as internal.  
- Smoke `smoke-finish.mjs` still references `client.encodeSeedMarket` (stale vs current `client.ts`) — rename pass should fix smokes, not revive encode\* on Client.

### Uniswap\*

- No `Uniswap*` on public barrel (good).  
- Still: package README lead (“Uniswap v4”), class comment “Uniswap v4 outcome-collateral pool”, type `QuoteExactInResult`, internal `src/uniswap/*` imported by barrel for `quoteSeed` / `LpPosition` (path not visible to FE).  
- Field names like `sqrtPriceX96` on `PoolState` are protocol-necessary; acceptable low-level Pool surface.

### Factory\*

- `listFactoryMarkets` exposes deployment registry concept. Prefer address-list wording; keep contract ABI/name internal.

---

## quoteX ↔ writeX pairing audit

| Quote / preview | Write | Status |
|-----------------|-------|--------|
| `pool.quoteTrade` | `pool.swap` | **Good** — keep as template |
| `quoteSeed` (standalone) | `market.seed` | **OK** — mild name asymmetry |
| *(none)* | `market.addLiquidityAcrossOutcomes` | **Gap** — no `quoteAddLiquidity` / size preview API (may be out of scope) |
| `estimatePriceAfterSwap` | — | **Orphan** — fold into `quoteTrade.priceAfter` |
| `isPriceAboveNinetyNineCents` | — | Gate helper; belongs as quote field only once renamed |
| — | `market.resolve` / `merge` | No quote needed |
| — | `client.createMarket` / `redeem` / `burn` | No quote needed |

**Do not** reintroduce public `encodeX` as the write twin; writes stay high-level (`swap`, `seed`, `addLiquidity`).

---

## positions vs lpPositions (deeper)

| API | Meaning today | FE usage |
|-----|---------------|----------|
| `user.positions` | Outcome **share** book (`UserSharePosition`) | `lib/holdings.ts` |
| `user.lpPositions` | Uniswap v4 **LP NFTs** | Profile + LiquidityPanel |
| `market.listLpPositions` | Enrich caller-supplied LP rows | Via User only |
| FE `marketKeys.positions` | React Query key for **LP** fetches | LiquidityPanel / profile invalidation |

**Problem:** three different “positions” meanings. Rename SDK share API to `sharePositions`; optionally FE key → `marketKeys.lpPositions` in a follow-up (FE-only, outside this SDK rename).

---

## displayPrices / displayMetrics vs outcomePrices

| Method | Source | FE |
|--------|--------|----|
| `displayPrices` | indexer → RPC | Market page, InvestPanel |
| `displayMetrics` | indexer → RPC | Market page |
| `outcomePrices` | RPC only | **unused** |

Canonical product names are already `display*`. Demote `outcomePrices` to private implementation detail of `displayPrices` (or `livePrices` if a debug escape hatch is required).

---

## Suggested migration order

1. **Types / unions with brand** (`Ponder*`, `source: "ponder"`, `Encode*Params`, `QuoteExactInResult`) — alias old names briefly if needed.  
2. **User share rename** (`positions` → `sharePositions`) before FE confuses more keys.  
3. **Pool gate + after-swap** collapse into `quoteTrade` fields; deprecate standalone methods.  
4. **`addLiquidityAcrossOutcomes` → `addLiquidity`** (single FE call site).  
5. **`listFactoryMarkets` → `listMarketAddresses`**.  
6. Demote `outcomePrices` / `listLpPositions` visibility.

Keep **compat aliases** for one minor version where FE blast > 0; drop aliases when FrontendSdk / frontend landed.

---

## Out of scope / keep as-is

- Class names `VoizMarketsClient`, `VoizMarket`, `VoizPool`, `VoizUser`, `VoizOutcome`  
- `normalizeOddsBps`, `quoteSeed`, `resolvedDisplayRows`, `marketImagePath`, `aggregateOutcomeCollateral` (helpers; separate dead-export audit)  
- Internal module folders `uniswap/`, `ponder/`, `encode/`  
- ABI / `sqrtPriceX96` / PoolKey shape on advanced Pool reads  

---

## Cross-links

- `_audit-duplicate-apis.md` — ownership / un-export of raw `fetch*FromPonder` and `*Calls`  
- `_audit-fe-dead-exports.md` — which barrel symbols FE actually imports  
- `_audit-react-types-abi.md` — React / ABI surface  

