---
name: sui-mrc20-assets
description: Analyze a Sui address's current Movescriptions/MRC20 inscription objects and tick-linked fungible coin balances, determine which inscriptions the published contract can burn, and distinguish current holdings from past mint/burn events.
---

# Sui MRC20 asset analysis

Use this skill when a user wants to find their current Movescriptions/MRC20 assets, understand why a wallet scanner shows zero, identify burnable inscriptions, or compare current holdings with prior protocol use.

## Rules that prevent false results

- Treat **current state** and **transaction history** as separate answers. A `MintTick`/`BurnTick` event is historical evidence, not a wallet asset. A past mint does not prove that an object or coin is still held.
- Identify tokens by their full on-chain Move type and the protocol registry, never by ticker, symbol, display name, or amount shape alone.
- Distinguish a `Movescription` object from a fungible `Coin<T>`. The protocol may support burning inscriptions without exposing a permissionless burn for converted coins.
- A failed query, incomplete pagination, RPC/indexer error, wrong network, or wallet/address mismatch is **unknown**, not “no assets.” State what could not be checked.
- Do not ask for or handle seed phrases/private keys. This workflow is read-only unless the user separately and explicitly asks to execute a burn.

## Workflow

1. **Resolve the account and network.** Use the address the user supplied. If using a connected wallet, confirm the displayed address and Sui network before querying. Normalize the address for queries, but preserve its original form in the report.
2. **Discover the live protocol deployment and type identities.** Read the project's current source/README or authoritative deployment registry, then verify relevant package/deployment objects on Sui mainnet. Do not copy package IDs or object IDs from an old conversation without rechecking them. Record the object type origin, deployment/registry object, and versioned entry-point package separately; Move type origin and latest call package can differ after upgrades.
3. **Enumerate current inscription objects.** Query all pages of objects owned by the address for each verified supported `Movescription` type. Inspect object contents for tick, amount, accumulated/locked SUI, and attached-coin/status fields. Resolve the object's tick to the correct current tick record. Do not claim an object is burnable from its symbol or type name alone.
4. **Enumerate current fungible balances.** Page through all `Coin<T>` balances for the address. Page through the protocol tick registry; for each tick record, read its `Treasury<T>` dynamic field and `coin_type`. Normalize only the address padding in the full type string, then match the exact full `Coin<T>` type to the wallet balance. Show the tick, full type, raw balance, and decimals/source when available. Never infer MRC20 membership from a ticker match.
5. **Determine burnability from deployed code.** Inspect the exact published Move entry point and its arguments/requirements for the object's package version. Mark an inscription burnable only when the owned object, matching tick record, attached-coin constraints, and entry-point compatibility are verified. Report any unsupported objects with the concrete reason. Treat `Coin<T>` balances as non-burnable unless the verified deployed code exposes a user-callable burn path and the user has the required capability; do not use a treasury capability from registry state as if the wallet owned it.
6. **Analyze history only when requested or needed to explain a discrepancy.** Paginate address-sent transactions for the user's actions; use affected-address history as a separate supplemental view for received/transferred objects. Inspect Move call targets, effects status, object changes, and protocol events such as `MintTick` and `BurnTick`. Deduplicate transaction digests and distinguish function invocations from successful effects. Trace created object IDs forward if the user asks what happened to past mints; do not infer disposition from event counts alone.
7. **Report with evidence and scope.** Give the query time, network, address, pagination/completeness, current inscription count, exact tick-linked coin matches, burnable count and reasons, and a separate historical summary. Include transaction digests or explorer links for material claims when useful. State limitations and unresolved types explicitly.

## Burn execution boundary

This skill's default output is an analysis, not a transaction. Do not construct, sign, submit, or batch a burn merely because an asset is marked burnable. If the user explicitly asks to burn, show the exact objects/ticks/amounts and estimated SUI return, explain irreversible effects and gas, then require the user's wallet to review and sign each intended transaction. Never request private keys or bypass wallet confirmation.

## Useful result format

| Asset | Kind | Current amount | Burnable | Reason |
|---|---|---:|---|---|
| TICK | inscription / `Coin<T>` | amount | yes / no | verified contract condition or unsupported path |

Follow with a short separate history section. Use “no matching assets found” only after all required current-state pages completed successfully; otherwise say the scan is incomplete.
