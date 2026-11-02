# 06. Merchant Dashboard

[DjedAlliance/StablePay-MerchantDashboard](https://github.com/DjedAlliance/StablePay-MerchantDashboard) is the app where a merchant reads the payments StablePay brought in. It is a Next.js (App Router) + TypeScript project with wagmi for the wallet and a static export for deployment. It reads payments directly from the chain: there is no indexer or backend, so every performance problem is an RPC problem.

Most of weeks 4 to 6, and week 20, went here.

## 1. A broken production build (PR #66)

Before starting feature work I found that the production build failed: four PRs merged earlier had left a missing component import. I fixed the import, the production warnings from missing libraries, and the theme toggle in the mobile header, so later feature PRs could be checked against a working build.

## 2. Development vs production merchant address (PR #67)

**Problem.** The dashboard only showed transactions for the configured merchant address. A developer working on a feature often had no transactions of their own, so they edited `merchant_address` to an empty string every time and had to remember to revert it.

**Change.** `npm run dev` now uses a development environment in which the merchant address is empty, so all transactions are fetched. The production build still connects the wallet and shows only that wallet's transactions. I recorded both modes in the PR for the reviewers.

## 3. Pagination, sorting and progressive fetching (PR #68, fixes #31)

The transactions page fetched everything and then rendered it, so the page stayed blank while it scanned the chain.

- **Pagination** with page sizes 25, 50, 100 and 200.
- **Sorting** by timestamp, block number, stablecoin amount and base-coin amount, ascending or descending.
- **Progressive fetching.** Logs are fetched in block ranges and transactions appear as each range arrives.
- **Lazy, grouped timestamps.** An event log carries a block number but not a timestamp; the timestamp needs one more RPC call per block. The dashboard fetches timestamps only when sorting by time, and once per block for all the transactions in it.
- **A 1,000-block cap** per fetch, with a "fetch more" button, so a first load is always bounded.
- Follow-up commits fixed a race condition between overlapping fetches and a bug where clearing the cache left stale rows on screen.

## 4. Caching (PRs #69 and #71, fixes #6)

The transaction cache expired after five minutes, so a merchant reopening the dashboard rescanned the chain.

- **#69** was the larger design: cache transactions in `localStorage` indefinitely, track the last synced block per network (Sepolia, ETC mainnet, Mordor), resume fetching from that block instead of from the deployment block, append new transactions and deduplicate by transaction hash.
- **#71** is the minimal change that solved the issue and was merged: the existing cache's TTL became infinite, so cached transactions persist across reloads and sessions until the user clears them. An earlier commit also aligned the stats cache's TTL with the transactions cache so the two can't disagree.

## 5. Filters, copy buttons and CSV export (PRs #72 and #73, fixes #30 and #32)

The filter button existed but did nothing.

**Filters.** A filter panel with the logic behind it. A merchant can filter by buyer address, receiver address, blockchain, block range, stablecoin amount range, base-coin amount range, risk level, status and date range.

**Copy buttons and info card.** A merchant who wants to filter by a buyer usually needs to copy that buyer's address from a transaction first. I added copy buttons to every field worth copying and improved the transaction info card, and explained in the PR why a UI change belonged in a filter PR.

**CSV export** through an export dialog:

- follows the current sort field and direction;
- lets the user choose whether to apply the active filters;
- lets the user choose which columns to include.

The filter work started as #72 and was combined with the export into #73, which was merged.

## 6. Multiple merchant addresses (PR #74)

A merchant may receive payments at more than one address (a separate address per store, or a hot and a cold wallet). The dashboard only showed the connected wallet's transactions.

- Merchants can now add EVM addresses manually in the sidebar and monitor all of them at once.
- The fetch layer queries every configured address and merges the results.
- A `use-wallet` hook separates the connected wallet from the list of monitored addresses.

## 7. UI/UX refresh (PR #75, open)

A cleaner, more professional fintech look that keeps the StablePay brand and puts the dashboard's core job (reading received payments) first:

- reworked global styles, layout, sidebar, mobile header and stat cards;
- a simpler transactions page and filter panel;
- a **text-size control** with a font-scale hook, so a merchant can enlarge the whole dashboard for readability.

## 8. Tectonic transactions (PR #76, open)

Once Tectonic was live on Sepolia, the dashboard needed to show Tectonic payments.

- A **Djed/Tectonic switch** on the overview and transactions pages. The selection is shared across pages and persisted.
- Djed keeps its existing deployments, cursors and cache keys, so nothing a Djed merchant had cached is lost.
- Tectonic payments are read from **`Minted` events** on the Sepolia deployment (`0xea262fB1…2aC1`), starting from block 12,027,502 (the deployment block), with 18-decimal stablecoin amounts.
- Transactions and overview stats are **cached per protocol**, and the network filter only lists the selected protocol's networks.
- **Archive history.** publicnode prunes `eth_getLogs` history and failed on both the Djed and the Tectonic Sepolia ranges. Sepolia now reads through **Tenderly's free public gateway**, which keeps full history and allows browser requests, with publicnode as a fallback if Tenderly errors.

## Summary

| PR | Feature | Status |
|---|---|---|
| #66 | Production build fix | Merged |
| #67 | Dev/prod merchant address | Merged |
| #68 | Pagination, sorting, progressive and lazy fetching | Merged |
| #69 → #71 | Permanent transaction cache | #71 merged |
| #72 → #73 | Filters, copy buttons, CSV export | #73 merged |
| #74 | Multiple merchant addresses | Merged |
| #75 | UI/UX refresh, text-size control | Open |
| #76 | Tectonic transactions | Open |
