# OutcomeAmm docs rewrite — writer notes (auditor)

Public developer docs now describe the live complete-set OutcomeAmm stack, not Uniswap v4 LP NFTs.

## Skill

`skill_resolution: paths-injected` — loaded `/home/s1x3l4/.claude/skills/cognitive-doc-design/SKILL.md`.

## Files rewritten (assigned)

| File | Change |
|------|--------|
| `sdkDev/sdk/README.md` | Client shape, `pool(token, ctx)`, seed/`addFunding`, no burn/rebalancer |
| `sdkDev/sdkdocs/index.mdx` | One AMM; share book + LP shares |
| `sdkDev/sdkdocs/concepts.mdx` | Invalid extra slot; indexer share book only |
| `sdkDev/sdkdocs/protocol.mdx` | `p_i ∝ 1/x_i`; fees to treasury; halt at resolve; no rebalancer |
| `sdkDev/sdkdocs/api/user.mdx` | Real `trades` / `sharePositions` / `lpPositions` DTOs |
| `sdkDev/sdkdocs/api/client.mdx` | `pool` ctx; no `client.burn` |
| `sdkDev/sdkdocs/guides/positions.mdx` | Share + LP event books; redeemed zero-share lots |
| `sdkDev/sdkdocs/guides/provide-lp.mdx` | `seed` / `addLiquidity` → `addFunding`; no SDK remove |
| `sdkDev/sdkdocs/guides/liquidity.mdx` | Inventory P&L; AMM fee is not LP income |
| `sdkDev/sdkdocs/guides/trade.mdx` | AMM `quoteTrade`/`swap`; one buy moves other named prices |
| `indexer/README.md` | Share/LP books, chart settlement pin, snapshots skip resolved |

## Extra public pages (still lied after the assigned list)

Patched so the Mintlify nav does not contradict the rewrite:

| File | Change |
|------|--------|
| `sdkDev/sdkdocs/guides/seed.mdx` | `addFunding` + odds hint; `quoteSeed` total = `seedWei` |
| `sdkDev/sdkdocs/api/market.mdx` | Seed = `addFunding`; `seedStatus` none/seeded |
| `sdkDev/sdkdocs/api/pool.mdx` | Required `{ market, outcomeIndex }`; OutcomeAmm swap |

Left alone as requested: `sdkDev/sdkdocs/_audit-*.md`, `bot/`, Uniswap v4 leftovers, permit2. `dump.mdx` is historical and not in `docs.json` nav.

## Facts encoded

1. One OutcomeAmm per market; named prices `p_i ∝ 1/x_i`; Invalid extra slot, not tradable, price 0.
2. `client.user(addr)`: `trades()` fills; `sharePositions()` indexer only; `lpPositions()` ERC-20 pool shares (one lot per market).
3. Share book: Buy/Sell + wrapped Transfer to Router after resolve. PnL = realized + mark. Redeemed lots kept when shares=0 and resolved.
4. LP book: FundingAdded/Removed + LP Transfer (skip mint/burn). Cost = max(amountsAdded). Remove proceeds = named basket at 1e18 (0/1 if resolved).
5. `/chart` pins winner 100 / losers 0. Snapshots skip resolved markets.
6. Trading is `client.pool(outcomeToken, { market, outcomeIndex }).quoteTrade/swap`.
7. No `client.burn`, no PositionManager NFTs. Add path: `market.seed` / `market.addLiquidity`.

## APIs that could not be verified as current SDK writes

- **`removeFunding` wrapper** — contract + ABI exist; no `market.removeLiquidity` / `client.removeFunding`. Docs say call the contract.
- **`seedStatus` `"partial"`** — still in the TypeScript union; implementation only returns `"none"` \| `"seeded"`.
- **`addLiquidity` leftover knobs** (`continueOnError`, `skipUninitialized`, `getSqrtPriceX96`, `outcomeLabels`) — still on the method; `encodeAddLiquidity` always sends one `addFunding` with `outcomes: []`.
- **`pool.priceFromState`** — returns `null` even when `status === "open"`. Not documented as a working mid.
- **`Trade.hookFee`** — field exists; `fetchTradesFromPonder` always sets `0n`.
- **VOIZ token LP rewards** — not on `OutcomeAmm`. Removed from public LP docs.
- **`quoteTrade.priceAfter`** — currently copied from spot, not a simulated post-trade mid.

No TypeScript was changed. No git commit.

## Revision (review pass)

Addressed all 10 open issues in `_amm-docs-review.md` (including nits). No wontfix.

| Issue | Fix |
| ----- | --- |
| 1 | Quickstarts use `pool(token, { market, outcomeIndex })` |
| 2 | `api/react.mdx` live `ChainAddresses` only |
| 3 | `api/helpers.mdx` real address overrides, no `hook` |
| 4 | Concepts: catalog RPC fallback vs portfolio `[]` |
| 5 | Invalid not in `market.prices().rows`; `OutcomePrices` on `api/market.mdx` |
| 6 | Share book: any wrapped transfer **to** AMM `router()`; proceeds = amount iff resolved winner |
| 7 | `prices` / `metrics` `{ source, rows }` on `api/market.mdx` |
| 8 | Indexer README lists `/v1` prices, LP, account routes |
| 9 | Pool `state` leads with `status` / `liquidity` / `poolId` |
| 10 | User API: one DTO block per method; related types as a table |
