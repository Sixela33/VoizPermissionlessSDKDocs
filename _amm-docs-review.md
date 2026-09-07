## Design Document Review: OutcomeAmm public docs

### Summary
**approve**

Re-verified the writer’s follow-up against live SDK, indexer, and `OutcomeAmm.sol`. All ten previous issues are closed. No new factual errors, invented APIs, or typecheck-failing samples on Mintlify nav pages. Public docs match the complete-set OutcomeAmm stack.

Verified: `quickstart-react.mdx`, `quickstart-viem.mdx`, `api/react.mdx`, `api/helpers.mdx`, `concepts.mdx`, `protocol.mdx`, `api/market.mdx`, `api/pool.mdx`, `api/user.mdx`, `guides/positions.mdx`, `indexer/README.md` against `client.ts`, `market.ts`, `pool.ts`, `types.ts`, `react/context.tsx`, `indexer/src/index.ts`, `indexer/src/api/v1.ts`.

---

### Strengths
- First-click path now uses `client.pool(token, { market, outcomeIndex })` in both quickstarts; React sample takes the required props.
- `ChainAddresses` on `api/react.mdx` matches `types.ts`. Helpers override with real keys (`marketFactory` / `router` / `collateral`); no `hook`.
- Concepts split catalog RPC fallback (`usedFallback`) from portfolio/history `[]`. Invalid is named-only on `market.prices().rows`; extra slot is `OutcomeAmm.prices()[numOutcomes] === 0`.
- Share book documents the live indexer rule: any wrapped transfer to AMM `router()` is an exit; proceeds = amount iff resolved winner. Open-market merge vs redeem is explicit.
- `api/market.mdx` `prices` / `metrics` `{ source, rows }` match `OutcomePrices` / `OutcomeMetrics`, including RPC `volumeCollateral: null`.
- Indexer README lists legacy REST and live `/v1` market, prices, LP, chart, and account routes (`v1.ts`).
- Pool `state` leads with `status` / `liquidity` / `poolId`. User API keeps one DTO per method.

### Out of scope
- `sdkDev/sdkdocs/_audit-*.md` and `dump.mdx` still describe LP NFTs / `client.burn`. They are not in `docs.json`.
