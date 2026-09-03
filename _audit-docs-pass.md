# Docs audit pass — Voiz Markets SDK (Mintlify)

**Mode:** read-only vs SDK truth (no docs/SDK content edits except this report).  
**When:** 2026-09-02 ~22:45 ART (UTC-3)  
**Docs:** `sdkDev/sdkdocs`  
**SDK:** `sdkDev/sdk/src` (barrel `index.ts`, `client.ts`, `market/market.ts`, `uniswap/seed-plan.ts`, …)  
**Nav:** `docs.json` · unpublished: `.mintignore` → `dump.mdx`, `_audit-*.md`

## Method

Compared every published MDX page + `dump.mdx` / prior `_audit-*` to the current public surface:

- Root: `sdk/src/index.ts`
- Client writes: `createMarket` / `redeem` / `burn` (no `seedMarket`, no public `send`, no `encode*`)
- Market: `summary` / `seed` / display helpers; `load` + `summaryFromRpc` are **private**
- Seed quote: `quoteSeed({ seedWei, oddsBps })` → totals only (not full `SeedPlan`)

Alexis return-shape rule checked as: **usage call first, then a full `// returns` object** (API refs mostly comply; guides often only destructure).

---

## Priority legend

| Tag | Meaning |
|-----|---------|
| **P0** | Wrong / documents removed or private APIs / will mislead integrators |
| **P1** | Stale inventory, bloat, voice, or incomplete return shapes |
| **P2** | Nice-to-have trim / polish |

---

## DELETE

| Priority | Item | Evidence | Why |
|----------|------|----------|-----|
| **P0** | `_audit-duplicate-apis.md` | File still says **KEEP BOTH** `client.seedMarket` + `market.seed`; tables list `encodeSeedMarket`, root `*Calls`, `previewSeedTotals`. SDK: no `seedMarket` on client (`client.ts`); sole write is `VoizMarket.seed` (`market.ts` ~470). | Contradicts shipped SDK; mintignored but still on disk — agents/humans will re-learn wrong truth. |
| **P0** | `_audit-fe-dead-exports.md` | Still lists FE imports of `previewSeedTotals`; checklists keep that name. SDK renamed → `quoteSeed` (`seed-plan.ts` ~77; dump.mdx night cleanup note). | Stale FE/export checklist. |
| **P0** | `_audit-react-types-abi.md` | Mentions `client.seedMarket`, `encodeSeedMarket`, `MarketSnapshot` KEEP BOTH + `load()`, `previewSeedTotals`. | Same vintage as pre-trim; superseded by this pass. |
| **P1** | `dump.mdx` (optional) | Already in `.mintignore` (not in `docs.json`). Accurate *as of* 2026-09-02 night cleanup, but overlaps every `/api/*` page + is a wall of changelog bullets. | **Either delete** once API pages are trusted, **or Keep mintignored** as living private inventory (see Keep). Do **not** publish. |

---

## FIX

| Priority | Item | Evidence | Action |
|----------|------|----------|--------|
| **P0** | `api/market.mdx` documents **`MarketSnapshot`** | § Related types (~L310–328). Type is **not** root-exported (`index.ts` explicitly un-exported per dump; lives only in `types.ts` / private `load`). Guide also teases it: `guides/resolve.mdx` L32. | Remove `MarketSnapshot` section; stop naming private `load` / `summaryFromRpc` in consumer docs beyond a one-liner “use `summary`”. |
| **P0** | `api/helpers.mdx` steers readers at private **`load`** | L308: “Prefer over raw on-chain `load` for UI.” | Rephrase to “prefer `summary` / `listMarkets` for UI” — never teach `load`. |
| **P1** | `quoteSeed` ≠ `SeedPlan` | `helpers.mdx` L256: “full plan returned by `market.seed` / **mirrored by `quoteSeed` totals** (`SeedPlan`)”. Truth: `quoteSeed` returns only `{ splitAmount, poolCollateral, totalCollateral }` (`seed-plan.ts` 77–94); `legs` only on `planMarketSeed` / `market.seed` → `plan`. | Fix copy: quote = totals; `SeedPlan` = `market.seed`’s `plan`. |
| **P1** | Guides miss full return shapes (Alexis) | `guides/seed.mdx` destructures seed result, no `// returns` block. `guides/provide-lp.mdx` burn call has **no** `{ result, hashes }` return. `guides/trade.mdx` comments fields but no full `QuoteExactInResult` object. API pages mostly OK. | Add compact `// returns { … }` after each primary call in guides (mirror `api/market` seed / `api/client` burn). |
| **P1** | Encode-named types on Market page | `EncodeResolveParams` / `EncodeMergeParams` documented (`api/market.mdx` ~371+) while `encodeResolve` / `encodeMerge` are **private** (`market.ts`). Types *are* root-exported via `market/index.ts`. | Rename headings to “`resolve` params” / “`merge` params” (or inline on those methods); drop “Encode*” product voice. |
| **P1** | Helpers type bloat / internal-ish types | `api/helpers.mdx`: `CreateMarketArgs`, `SeedOutcomeParams`, `V4PoolKey` full dumps. Still root-exported, but product path doesn’t need them. | Demote to one-line “advanced / internal” or drop sections; keep `CreateMarketParams`, `RedeemParams`, `MarketSummary`, `quoteSeed` types. |
| **P1** | Voice: **DTO** / walls | “DTO” in `api/market.mdx` L19/54, `helpers.mdx` L308, `dump.mdx`, SDK JSDoc. Guides are punchier (“Spin up…”) vs API jargon. `helpers.mdx` ~390 lines is a type encyclopedia. | Prefer “summary object” / “list row”; shorten helpers; one idea per sentence (AGENTS.md). |
| **P2** | `api/client.mdx` class sketch omits `sendCalls` | Class block L15–22 shows `publicClient` / `walletClient` only; `ClientConfig` has `sendCalls`. Runtime field exists (`client.ts` L89). | Add `readonly sendCalls?` or point only at config (avoid implying public `client.send`). |
| **P2** | Asymmetric write returns undocumented as pattern | `createMarket` / `redeem` / `burn` → `{ result, hashes }`; `market.resolve` / `merge` / `pool.swap` → bare `unknown`. Docs match code but never say why. | One Tip on Client or Concepts: “client-level writes return hashes; market/pool sends return opaque sender result.” |
| **P2** | Undocumented `market.outcome()` | Public on `VoizMarket` (`market.ts` ~335); not in `api/market.mdx`. | Document briefly or explicitly omit as thin `VoizOutcome` facade. |

### Explicit check — removed / private APIs in **published** docs

| Name | Published docs status | SDK truth |
|------|----------------------|-----------|
| `load` | Mentioned as private / “prefer summary” (`api/market` L8; helpers L308 bad) | `private async load` |
| `summaryFromRpc` | Mentioned private (`api/market` L8) | `private` |
| `client.seedMarket` | Correctly denied (`guides/seed`, `concepts`) | Removed |
| `encode*` public methods | Not taught as callable; types still “Encode*” | Private methods / action helpers |
| `client.send` / public submit | Not documented as API; `submitCalls` `@internal` | OK |
| `poolKey` on **burn** | Correct: `outcomeToken` (+ optional collateral) (`api/client` burn, `guides/provide-lp`) | `burn` builds key internally (`client.ts` 256–283) |
| `display*` vs wrong names | Prefer `displayPrices` over `outcomePrices` — consistent | OK |
| `previewSeedTotals` | **Absent** from published MDX (only stale `_audit-*`) | Renamed `quoteSeed` |
| `quoteSeed` / `market.seed` / `summary` | Guides + API match current signatures | OK |

---

## MERGE

| Priority | Item | Evidence | Suggestion |
|----------|------|----------|------------|
| **P1** | Trim `api/helpers.mdx` into thinner pages | Overlaps type blocks already inlined under Client/Market/Pool/User returns. | Keep helpers for **values** only (`getAddresses`, `quoteSeed`, `normalizeOddsBps`, `resolvedDisplayRows`, `marketImagePath`, `aggregateOutcomeCollateral`, `erc20Abi`). Move/`link` product types beside the methods that return them. |
| **P2** | `concepts.mdx` ↔ `index.mdx` mental model | Both teach Client → Market → Pool → User. | Keep both; shorten index “Mental model” to 5 lines + link concepts (already mostly that). |
| **P2** | Resolve guide tip vs Client/Market split | `guides/resolve.mdx` table already maps resolve/merge vs redeem/burn. | No merge needed; optional cross-link from `api/client` writes. |

Do **not** merge the six guides into one mega-page — each maps a product verb and stays short.

---

## KEEP

| Item | Evidence | Notes |
|------|----------|-------|
| Nav IA (`docs.json`) | Get started → Guides → API | Sensible; less-docs-friendly already. |
| Seed story | `guides/seed.mdx`, `concepts` seed table, `quickstart-viem` seed, `api/market` seed + `api/helpers` quoteSeed | Matches SDK: no `client.seedMarket`; `quoteSeed` then `market.seed`. |
| Burn without `poolKey` | `api/client` burn + `guides/provide-lp` | Matches `client.burn({ tokenId, outcomeToken, recipient })`. |
| Trade path | `guides/trade`, `api/pool` quoteTrade/swap | Correct vs `displayPrices`. |
| Catalog path | `listMarkets` / `summary` / `MarketSummary` shapes | Return objects on API pages match `ponder/types.ts` + `ListMarketsResult`. |
| React surface | `api/react.mdx` vs `react/index.ts` | Provider + `useVoiz` + types aligned. |
| `.mintignore` for audits/dump | Present | Correct — keep unpublished until deleted. |
| `dump.mdx` (alt) | Private inventory still matches post-trim barrel | **Keep mintignored** *only* if you want a single checklist of every export; otherwise Delete (see above). Prefer one source of truth: published `/api/*`. |

---

## dump.mdx / `_audit-*` relevance (Q6)

| File | Published? | Still relevant? | Verdict |
|------|------------|-----------------|--------|
| `dump.mdx` | No (`.mintignore`) | Partially — good snapshot of **current** public API + trim changelog | **Keep mintignored** as eng inventory **or Delete** after API pages own the truth. Do not add to `docs.json`. |
| `_audit-duplicate-apis.md` | No | **No** — pre-trim recommendations | **Delete** |
| `_audit-fe-dead-exports.md` | No | **No** — `previewSeedTotals` era | **Delete** |
| `_audit-react-types-abi.md` | No | **No** — same | **Delete** |
| `_audit-docs-pass.md` (this file) | No (matches `_audit-*.md`) | Yes — actionable docs backlog | Keep until fixes land, then delete or archive |

---

## Suggested fix order

1. Delete three stale `_audit-*.md` (duplicate / fe-dead / react-types).  
2. Strip `MarketSnapshot` + private-`load` teaching from `api/market.mdx` + `api/helpers.mdx` (+ resolve guide comment).  
3. Fix `quoteSeed` vs `SeedPlan` wording in helpers.  
4. Add Alexis `// returns` blocks to seed / provide-lp / trade guides.  
5. Soften DTO/Encode* voice; slim helpers.  
6. Decide dump.mdx: keep private or delete.

---

## Top findings (executive)

1. Prior `_audit-*.md` files are **dangerously stale** (still KEEP `client.seedMarket` / `previewSeedTotals`).  
2. Published API wrongly **documents `MarketSnapshot`** (not root-exported).  
3. Helpers still point UI authors at private **`load`**.  
4. **`quoteSeed` mislabeled** as mirroring full `SeedPlan`.  
5. Guides **skip full return objects** (Alexis rule) for seed / burn / quoteTrade.  
6. **Encode\*** type names imply removed public encode APIs.  
7. **`dump.mdx`** useful privately, not for Mintlify nav; overlapping with `/api/*`.  
8. Burn / seed / summary / quoteSeed **examples in guides are otherwise current** (no `poolKey` on burn).  
9. Voice: **DTO** + helpers type-wall vs punchy guides.  
10. Less-docs win: delete stale audits, slim helpers, don’t publish dump.
