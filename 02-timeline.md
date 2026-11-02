# 02. Timeline

This file has two views of the project. The first is the week-wise plan as it was tracked in my weekly goals sheet and reported to mentors. The second is a dated chronology taken from commit and pull-request history. The two don't line up week for week: some goals were finished early, some took longer than planned, and the Tectonic work changed direction after the mid-term evaluation. Both views are accurate. The plan shows what each week was *for*. The chronology shows *when* the work actually landed.

---

## Part A. Week-wise progress (goals sheet)

### Community bonding

**Task:** Revise the timeline and understand Tectonic.

- Revised the proposed timeline. The original plan included supporting several Djed variants; I dropped that in favour of studying Tectonic, since the organization had decided StablePay would move to it.
- Got the PRs from the preparation phase reviewed.
- Reviewed open issues and pull requests across the StablePay repositories.
- In parallel, contributed to the OrbOracle repositories (an automated price-poster service and fixes to the OrbOracle frontend). See [10-community-bonding-and-preparation.md](10-community-bonding-and-preparation.md).

### Week 1: Set up repositories and environment; review issues and PRs

- Set up all StablePay repositories (SDK, merchant dashboard, landing page) and the local development environment.
- Reviewed open issues and pull requests to pick the ones that fit the plan for weeks 2–6.

### Week 2: Improve UI/UX of the widgets; prevent unexpected behaviour (StablePay #105)

- Added a **React + Vite example website inside `stablepay-sdk`** that picks up every local SDK build. The existing demo site only updated when a new version was published to npm, so testing a widget change meant publishing first.
- Improved the design and responsiveness of `PricingCard`, `Dialog` and `TransactionReview`.
- Strengthened validation in the transaction flow. Fixes issues #60, #61 and #62.
- Fixed a freeze when the user has no wallet installed.

### Week 3: Better error handling; move the widgets to a bright theme (StablePay #105, more commits)

- More robust error handling and clearer feedback during transactions, including the no-wallet and logged-out-wallet cases.
- Moved the widgets to a bright theme, replaced the logo and removed an unnecessary confirm step from the flow.

### Week 4: Performance upgrades in the merchant dashboard (Dashboard #67, #68, #71)

- **Pagination** (25/50/100/200 per page) and **sorting** by timestamp, block number, stablecoin amount and base-coin amount, ascending or descending.
- **Progressive fetching:** transactions appear as they are fetched instead of after the whole scan.
- **Lazy timestamps:** block timestamps need an extra RPC call per block, so they are only fetched when sorting by time, and once per block for all transactions in it.
- A **1,000-block cap** per fetch with a "fetch more" button (fixes #31).
- **Separate merchant-address settings for development and production**, so a developer sees all transactions locally without editing config each time.
- **Non-expiring local transaction cache** (fixes #6).

### Week 5: Export with column choice and filters; working filter button (Dashboard #73)

- Implemented the filter logic behind the existing filter dialog.
- Added copy buttons for addresses and other fields, and improved the transaction info card, so a merchant can copy a buyer address straight into the filter.
- **CSV export** that follows the current sort order, optionally applies the active filters, and lets the user choose columns (fixes #30 and #32).

### Week 6: Multiple merchant addresses; responsiveness and UI/UX (Dashboard #74, #75)

- Merchants can add and monitor several EVM addresses at once instead of only the connected wallet.
- Refreshed the dashboard into a cleaner fintech look focused on reading received payments.

### Week 7: Read the Tectonic paper; set up Tectonic locally; list the functions StablePay needs

- Studied the Tectonic paper (June 2026 draft) section by section: prices, reserve ratio thresholds, stability fees, triggered redemptions.
- Set up `Tectonic-EVM-Contracts` locally and read the contract against the paper.
- Mapped every Djed call StablePay makes to its Tectonic equivalent and wrote down what was missing. See [08-research.md](08-research.md).

### Week 8: Deploy Tectonic locally; build and test a Tectonic adapter

- Found that the upstream contract compiled and deployed but could not be used: the very first `mint()` reverted.
- Built a patched local copy (`tectonic-local`) with a mock oracle, a deploy script and a test suite, and deployed it on Anvil.
- Built and tested the first version of the StablePay Tectonic adapter against it. See [09-tinkering-and-experiments.md](09-tinkering-and-experiments.md).

### Week 9: Improve the Tectonic adapter (StablePay #106)

- New `tectonic-sdk` package: pure pricing math plus a viem client.
- `ProtocolAdapter` interface with `TectonicAdapter` and `DjedAdapter`; `Transaction.js` delegates to whichever adapter the network config names.
- A `protocol` field in the network config, defaulting to Djed so nothing existing changed.
- The `tectonic-local` Foundry harness: patched contract, mock oracle, deploy script, 21 tests.

### Week 10: Remove Djed from `stablepay-sdk` (StablePay #107)

- Removed all three Djed integrations (Sepolia, Milkomeda, Ethereum Classic) and the `djed-sdk` dependency. This PR was later superseded by #110, which re-applied the same removal on top of the current `main`.

### Week 11: Tectonic PRs; mid-term recommended files and brand folder (Tectonic #22, StablePay #108)

- Fixed Tectonic defects found during integration: a nested `nonReentrant` guard that made `redeem()` and every triggered redemption revert, a recursive `chargeStabilityFee` that ran out of gas, and an `internal` `ecPrice()` that blocked off-chain integration.
- Added the repository documents and the `brand/` folder recommended at the mid-term evaluation.

### Week 12: Take over Tectonic after mid-term; test and submit improvements (Tectonic #24, #25)

- Seeded both coin supplies at deployment with unowned supply so ratio denominators can never be zero, and guarded forced redemptions against an empty holder set.
- Handled the zero-equity state (`R == L`) so views report the state instead of panicking, then reworked it with a **super-critical reserve ratio** that floors the ratio so equity never reaches zero.
- Fixed the gas-diff CI workflow and the Slither findings.

### Week 13: Revamp the website; get reviews (Landing Page #82)

- Replaced the dark theme with a bright design built from the StablePay and Tectonic palettes.
- Added five sections (payment flows, code, protocol, networks, FAQ), the real logo mark and Tectonic-only copy.

### Week 14: Deploy Tectonic on Sepolia (Tectonic #27)

- Wrote a Chainlink ETH/USD **inverting oracle** and a deployment script, with tests for both.
- Deployed and source-verified Tectonic, EquityCoin and the oracle on Sepolia, and recorded the deployment in `Deployments.md`.

### Week 15: Integrate the deployed Tectonic; remove `tectonic-local` (StablePay #111)

- Switched StablePay to the Sepolia deployment and removed the local Tectonic copy.
- Updated `tectonic-sdk` to the deployed ABI (dev fee instead of treasury fee).

### Week 16: Implement mentor feedback on the website (Landing Page #82)

- Two rounds of feedback from Bruno: content, terms of use, social links, copy, line breaks, network list.

### Week 17: Add StablePay documentation to the website (Landing Page #83)

- A `/docs` section: introduction, quick start, how a payment works, Tectonic, integration guides (any website, Next.js), payment verification, customization, widget API, Tectonic SDK, networks, troubleshooting and a glossary.

### Week 18: Record video tutorials

- Recorded tutorials showing how to integrate StablePay into a merchant website, and added them to the StablePay website.

### Week 19: Collect learning resources in one place

- Gathered resources scattered across the web (Djed, Gluon, OrbOracle and others) into one section of the StablePay website for new contributors.

### Week 20: Add Tectonic transactions to the merchant dashboard (Dashboard #76)

- A Djed/Tectonic switch on the overview and transactions pages. Sepolia history is read through Tenderly's public gateway, with publicnode as a fallback.

### Week 21: Review other contributors' PRs; get remaining PRs merged

- Reviewed other contributors' pull requests and followed up on my open ones.

### Week 22: Debugging and troubleshooting

- Tested StablePay end to end on Sepolia and fixed what I found. The main fix: StablePay now **verifies the amount actually minted** before reporting a payment as complete, because `mint(receiver)` has no minimum-output parameter.

---

## Part B. Dated chronology

| Date (2026) | What happened |
|---|---|
| May 3 | OrbOracle-EVM-Frontend #39 merged: decimal-scaling fix found while testing the price poster |
| May 12 | OrbOracle-Poster #2 merged: automated multi-source price poster service |
| May 16, 27 | OrbOracle-Poster #5 (README) and #6 (multi-coin support) merged |
| May 20 | PictoPy #1269 opened: watcher-service deadlock fix |
| May 26 | OrbOracle-EVM-Frontend #40 opened: corrected ETC factory address (merged Oct 8) |
| Jun 7–8 | Example website, widget UI/UX and validation commits; StablePay #105 opened |
| Jun 14, 19 | Bright theme, new logo, no-wallet and logged-out-wallet fixes pushed to #105 |
| Jun 21 | Dashboard #66: fixed a broken production build (missing import) |
| Jun 23–24 | Dashboard #67 dev/prod merchant address, merged |
| Jun 24–25 | Dashboard #68 pagination and sorting, merged |
| Jun 24–26 | Dashboard #69 persistent caching, replaced by the smaller #71 (merged Jun 26) |
| Jun 25–29 | Dashboard #72 filters, folded into #73 filters + CSV export (merged Jun 29) |
| Jun 30 – Jul 18 | Dashboard #74 multiple merchant addresses, merged Jul 18 |
| Jul 19 | Dashboard #75 UI/UX refresh opened. Wrote the Tectonic assessment and integration plan |
| Jul 19–25 | Built `tectonic-local`, `tectonic-sdk` and the adapter layer. Wrote the local-fork report and the question list for maintainers |
| Jul 25–27 | StablePay #106 (Tectonic adapter) opened |
| Jul 26 | StablePay #105 merged |
| Aug 10 | Fixed issues from Atharva's peer review on #106. Opened #107 (remove Djed). Opened Tectonic #22 |
| Aug 11 | Tectonic #22 merged |
| Aug 12 | StablePay #108 (mid-term recommended files, brand folder) opened |
| Aug 26–27 | Website revamp (#82) opened. Tectonic `fix-1-fresh` (#24) and `fix-4-6-fresh` (#25) opened |
| Aug 31 | Tectonic #24 merged |
| Oct 7 | #25 reworked around `superCriticalReserveRatio`. Hardening commits on #108: exact quotes, stale-write guard, error codes, local-fork patches 10–12, fee-scale validation, brand script tests |
| Oct 8 | StablePay #106 merged |
| Oct 17 | #25 CI and Slither fixes. StablePay #109 (sync `tectonic-local` with upstream) merged. #110 opened. First round of website feedback |
| Oct 18 | Tectonic #25 merged. StablePay #110 (Tectonic only) merged |
| Oct 21 | Tectonic #26 (dev, devFeeReceiver, devFee) opened, merged Oct 23 |
| Oct 30 | Tectonic deployed and verified on Sepolia (#27). `tectonic-sdk` matched to the deployed ABI; `tectonic-local` removed |
| Oct 31 | StablePay #111 opened, with mint verification. Second round of website feedback; #82 merged. Documentation PR #83 opened |
| Nov 1 | Dashboard #76 (Tectonic transactions) opened |
