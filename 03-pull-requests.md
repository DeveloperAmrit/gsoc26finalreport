# 03. Pull Requests

Every pull request I opened from community bonding to the end of the coding period. Sizes are lines added / removed as GitHub reports them (they include rebuilt `dist/` bundles and lock files where relevant, so they overstate the hand-written change for SDK PRs).

## Summary

| Repository | Opened | Merged | Open | Closed (superseded) |
|---|---|---|---|---|
| DjedAlliance/StablePay | 7 | 4 | 2 | 1 |
| DjedAlliance/StablePay-MerchantDashboard | 10 | 6 | 2 | 2 |
| DjedAlliance/StablePay-LandingPage | 2 | 1 | 1 | 0 |
| StabilityNexus/Tectonic-EVM-Contracts | 5 | 4 | 1 | 0 |
| **Coding period total** | **24** | **15** | **6** | **3** |
| Community bonding (OrbOracle, PictoPy) | 6 | 5 | 0 | 1 |

"Superseded" means the work was carried forward in a later PR: #107 → #110, #69 → #71, #72 → #73.

---

## DjedAlliance/StablePay

| PR | Title | Status | Size |
|---|---|---|---|
| [#105](https://github.com/DjedAlliance/StablePay/pull/105) | feat: add example website and improve SDK widget UI/UX & validation flow | Merged Jul 26 | +3,650 / −334, 34 files |
| [#106](https://github.com/DjedAlliance/StablePay/pull/106) | Add tectonic adapter | Merged Oct 8 | +3,718 / −184, 43 files |
| [#107](https://github.com/DjedAlliance/StablePay/pull/107) | Removed djed from stablepay | Closed, superseded by #110 | +261 / −16,383, 42 files |
| [#108](https://github.com/DjedAlliance/StablePay/pull/108) | Mid term fixes | Open | +6,678 / −195, 87 files |
| [#109](https://github.com/DjedAlliance/StablePay/pull/109) | chore(tectonic): sync tectonic-local with Tectonic-EVM-Contracts | Merged Oct 17 | +581 / −512, 14 files |
| [#110](https://github.com/DjedAlliance/StablePay/pull/110) | refactor: remove Djed support; StablePay uses Tectonic only | Merged Oct 18 | +266 / −16,390, 42 files |
| [#111](https://github.com/DjedAlliance/StablePay/pull/111) | fix: Use tectonic deployed on sepolia testnet | Open | +335 / −1,345, 29 files |

### #105: Example website, widget UI/UX and validation

Fixes #60, #61, #62.

- **Example website inside the SDK.** A React + Vite app under `stablepay-sdk/example/`. A separate demo repository already existed, but it pulled the SDK from npm, so it only showed changes after a release. The in-repo example picks up every local build, which made the rest of the project's widget work testable in seconds.
- **Widget redesign.** Restyled `PricingCard.css`, refactored `Dialog.jsx` and `TransactionReview.jsx`, improved responsiveness; later moved everything to a bright theme with a new logo and removed a redundant confirm step.
- **Validation and flow.** Reworked the payment flow around explicit interaction states; better wallet-missing, network-validation and error messages; fixed a freeze when the user has no wallet; fixed the logged-out-wallet case; clearer detection of connection/DNS failures and reverted transactions.
- Rebuilt `dist/esm` and `dist/umd`.
- Demonstrated with recordings for both "user has a wallet" and "user has no wallet".

### #106: Tectonic support behind a protocol adapter

Tectonic was not deployed anywhere when I wrote this, so the PR adds the integration layer and a local harness to develop against without touching the working Djed paths.

- **`tectonic-sdk/`**, a new package: `pricing.js` (pure pricing math that mirrors the contract) and `tectonic.js` (a viem-based `TectonicClient`). Ships source, no build step.
- **`stablepay-sdk/src/core/adapters/`**: a `ProtocolAdapter` interface with `TectonicAdapter` and `DjedAdapter`. `Transaction.js` delegates to the adapter named by the network config and keeps its public method names, so the widget barely changed.
- **`utils/config.js`**: a `protocol` field per network, defaulting to `"djed"`.
- **`tectonic-local/`**: Foundry project with a patched copy of the upstream contract, a mock oracle, a deploy script and 21 tests.
- Design points: the Tectonic contract is itself the stablecoin; fees scale by 1e18 not 1e24 and a mis-scaled fee throws; quotes round up and are refreshed just before signing; gas is estimated, not hard-coded at 500k, because `mint()` can run up to 100 forced redemptions first.
- Also fixed a display bug where the success callback reported the stablecoin invoice with the payment token's symbol ("5 ETH" for a 0.00255 ETH payment).
- Testing: 21 contract tests (two fuzz suites at 512 runs), 28 SDK unit tests, 14 end-to-end checks against a local deployment, and both payment paths driven manually through the widget on Anvil.
- Peer-reviewed by Atharva Naik; I fixed the issues he found in the pricing code and the review screen before merge.

### #107: Remove Djed

Removed all three Djed integrations from StablePay. It sat open while the Tectonic adapter was reviewed and went stale against `main`, so I re-applied it cleanly as #110.

### #108: Mid-term fixes

Requested at the mid-term evaluation.

- Repository documents: `CONTRIBUTING.md`, `SECURITY.md`, `MAINTAINERS.md`, `LICENSE`, a best-practices checklist with a JSON status file, an OpenSSF Scorecard workflow, and a rewritten `README.md`.
- A `brand/` folder: `Brand.md`, colour palette (CSS, JSON, SVG), typography, logo and mono logo, favicons and web manifest, and a Python script that generates every raster size from the SVG, with its own unit tests.
- Follow-up hardening commits:
  - an exact Djed quote (the 6-decimal display string had been used to build the payment and could underpay by up to 1e12 wei);
  - a cancellation flag so a superseded quote run cannot overwrite a newer one;
  - wallet error codes 4001 (user rejected) and −32005 (rate limited) no longer reported as connection failures;
  - the local network hidden unless explicitly enabled;
  - packaging fixes (`djed-sdk` external, `tectonic-sdk` bundled);
  - a narrower Vite `fs.allow`;
  - fee-scale validation in every pricing function;
  - three more fixes in the local Tectonic copy (patches 10–12, see [09](09-tinkering-and-experiments.md)).

### #109: Sync `tectonic-local` with upstream

After my upstream fixes merged, the local fork's patches were no longer needed.

- `tectonic-local/src/{Tectonic,Coin,IOracle,Math}.sol` made byte-identical to `Tectonic-EVM-Contracts` at `fix-4-6-fresh`. `MockOracle.sol` is the only local contract left.
- Picked up the super-critical reserve ratio and the seven-argument payable constructor that seeds unowned supplies.
- Deploy script updated (10 ETH seed); test suite rewritten for the synced contract (32 tests), including three `UPSTREAM_BUG` tests that pin defects still present upstream.
- `tectonic-sdk`: exposes `superCriticalReserveRatio` and `seedSupply`; `toBaseUnits` accepts a JS number only if it is a safe integer, so a float-rounded amount can no longer be quoted; tests run on Node 22+.
- `tectonic-sdk` now refuses to build a mint that is certain to revert below the critical ratio, instead of handing the payer a transaction that would only waste gas.

### #110: Tectonic only

Re-applies the Djed removal from #107 on current `main`.

- Deleted `djed-sdk/`, `DjedAdapter`, `ProtocolAdapter` and `createAdapter`. `Transaction` constructs `TectonicAdapter` directly and rejects the old `(uri, address)` form with a clear error.
- `utils/config.js` lists Tectonic deployments only.
- Dropped the `djed-sdk` and `web3` dependencies and their Rollup externals; rebuilt lock files and `dist`.
- Example app talks to Tectonic and shows setup steps when no address is configured. README describes the Tectonic SDK.

### #111: Use the Tectonic deployment on Sepolia

- `tectonic-sdk` matched to the deployed ABI: `treasury`/`treasuryFee` became `dev`, `devFeeReceiver`, `devFee`; `devFee` is now re-read on every quote because the dev can change it.
- Registered the Sepolia deployment (`0xea262fB18992ddE398a5b6fa7A4F879FfB262aC1`) as StablePay's network; dropped the local network, the localhost chain and `useLocalTectonic()`; pointed Sepolia at the publicnode RPC because viem's default rejects free requests.
- Replaced the local smoke test with `smoke-sepolia.mjs`, read-only by default with an optional real mint.
- Removed `tectonic-local` entirely.
- **Mint verification:** decode the `Minted` event from the receipt and only call `onTransactionComplete` when the merchant received at least the invoice. Otherwise the widget reports an underpayment and calls a new optional `onTransactionUnderpaid` callback with invoiced, received and shortfall amounts.

---

## DjedAlliance/StablePay-MerchantDashboard

| PR | Title | Status | Size |
|---|---|---|---|
| [#66](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/66) | fix: added missing component import | Merged Jun 21 | +17 / −0 |
| [#67](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/67) | fix: Separate merchant address settings for dev and prod | Merged Jun 24 | +13 / −15 |
| [#68](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/68) | feat: Add pagination and sorting to Transactions page | Merged Jun 25 | +871 / −118 |
| [#69](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/69) | fix: Added persistent caching | Closed, replaced by #71 | +137 / −54 |
| [#71](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/71) | fix: Made persistent caching infinite | Merged Jun 26 | +1 / −3 |
| [#72](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/72) | Feat: Added filter functionality | Closed, folded into #73 | +489 / −62 |
| [#73](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/73) | feat: add filtering and transaction export to CSV | Merged Jun 29 | +876 / −96 |
| [#74](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/74) | Feature: Multi-Merchant Address Support | Merged Jul 18 | +214 / −43 |
| [#75](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/75) | Improved merchant dashboard UI/UX | Open | +423 / −311 |
| [#76](https://github.com/DjedAlliance/StablePay-MerchantDashboard/pull/76) | feat: Add option to view tectonic's transactions | Open | +325 / −105 |

- **#66** fixed a failing production build (a missing import) and production warnings from missing libraries, as a follow-up to four earlier merged PRs.
- **#67**: `npm run dev` clears the merchant address so all transactions are fetched; the production build still shows only the connected wallet's transactions.
- **#68** (fixes #31): page sizes 25/50/100/200; sort by timestamp, block number, stablecoin and base-coin amounts; progressive fetching; timestamps fetched lazily and per block; a 1,000-block cap with "fetch more".
- **#69 → #71** (fixes #6): #69 introduced a persistent cache with per-network `lastSyncedBlocks` and incremental fetching from the last synced block, with deduplication by transaction hash. #71 replaced it with the minimal change that solves the issue: the existing cache's TTL became infinite.
- **#72 → #73** (fixes #30, #32): filter logic, copy buttons, a better info card, and CSV export that follows sort order, optionally applies filters and lets the user pick columns.
- **#74**: merchants add and monitor multiple EVM addresses instead of only the connected wallet.
- **#75**: a cleaner, more professional fintech look focused on reading payments.
- **#76**: a Djed/Tectonic switch; Tectonic payments read from `Minted` events on the Sepolia deployment from block 12,027,502; per-protocol caches; Sepolia read through Tenderly's public gateway with publicnode as fallback.

Details are in [06-merchant-dashboard.md](06-merchant-dashboard.md).

---

## DjedAlliance/StablePay-LandingPage

| PR | Title | Status | Size |
|---|---|---|---|
| [#82](https://github.com/DjedAlliance/StablePay-LandingPage/pull/82) | Revamped StablePay Website to a bright look | Merged Oct 31 | +1,570 / −585, 31 files |
| [#83](https://github.com/DjedAlliance/StablePay-LandingPage/pull/83) | feat: Add docs | Open | +6,248 / −73, 57 files |

- **#82**: replaced the dark space theme with a bright design built on the StablePay logo colours and Tectonic's brand; five new sections; Tectonic-only copy; responsive checks at 12 widths from 320 to 1920 px; two rounds of mentor feedback.
- **#83**: a 16-page documentation site at `/docs` with search, a sidebar, a table of contents, sequence diagrams and six interactive widgets, including live reads from the Sepolia deployment.

Details are in [07-website-and-documentation.md](07-website-and-documentation.md).

---

## StabilityNexus/Tectonic-EVM-Contracts

| PR | Branch | Status | Change |
|---|---|---|---|
| #22 | `fix-2-5-8` | Merged Aug 11 | `Tectonic.sol` +22 / −6 |
| #24 | `fix-1-fresh` | Merged Aug 31 | `Tectonic.sol`, `Coin.sol`, `Deployments.md` +60 / −2 |
| #25 | `fix-4-6-fresh` | Merged Oct 18 | `Tectonic.sol`, `Deployments.md`, CI workflow, `ChainlinkOracle.sol` |
| #26 | `dev-fee-receiver` | Merged Oct 23 | `Tectonic.sol`, `Deployments.md` |
| #27 | `deployment/testnet-sepolia-1` | Open | Inverting oracle, deploy script, tests, `Deployments.md` (+300) |

- **#22**: nested `nonReentrant` made `redeem()` and all triggered redemptions revert; `chargeStabilityFee` recursed until out of gas (~1.06 billion gas, then revert) whenever a fee was owed; `ecPrice()` was `internal`.
- **#24**: seed both supplies at deployment with coins owned by nobody, so `L()` and the supplies can never be zero; guard forced redemptions against an empty holder set.
- **#25**: first handled the zero-equity state in views and equity-coin minting; then, after mentor discussion, replaced that with a `superCriticalReserveRatio` that floors the reserve ratio so equity never reaches zero; added constructor validation; fixed the gas-snapshot CI job and addressed Slither findings.
- **#26**: replaced `treasury`/`treasuryFee` with a `dev` role, a `devFeeReceiver` and a mutable `devFee`, with a two-step dev handover and a `fee + devFee < 100%` check.
- **#27**: `ChainlinkInvertingOracle`, `DeployTectonic.s.sol`, 11 new tests, and the Sepolia deployment record.

Details are in [05-tectonic-contracts.md](05-tectonic-contracts.md).

---

## Community bonding

| PR | Title | Status |
|---|---|---|
| [StabilityNexus/OrbOracle-EVM-Frontend #39](https://github.com/StabilityNexus/OrbOracle-EVM-Frontend/pull/39) | Fixed decimal scaling issue | Merged May 3 |
| [StabilityNexus/OrbOracle-Poster #2](https://github.com/StabilityNexus/OrbOracle-Poster/pull/2) | feat: implement initial automated OrbOracle poster background service | Merged May 12 |
| [StabilityNexus/OrbOracle-Poster #5](https://github.com/StabilityNexus/OrbOracle-Poster/pull/5) | Update readme | Merged May 16 |
| [StabilityNexus/OrbOracle-Poster #6](https://github.com/StabilityNexus/OrbOracle-Poster/pull/6) | Added multi-coin support | Merged May 27 |
| [StabilityNexus/OrbOracle-EVM-Frontend #40](https://github.com/StabilityNexus/OrbOracle-EVM-Frontend/pull/40) | Changed address of ETC deployment | Merged Oct 8 |
| [AOSSIE-Org/PictoPy #1269](https://github.com/AOSSIE-Org/PictoPy/pull/1269) | Fix: Resolve watcher service deadlock during self-restart on folder deletion | Closed |

Details are in [10-community-bonding-and-preparation.md](10-community-bonding-and-preparation.md).
