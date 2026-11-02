# 01. Project Overview

## What StablePay is

StablePay is a decentralized payment solution for online merchants. When the StablePay widget is embedded in a website, it interacts directly with smart contracts on the blockchain. There is no backend server, no payment processor and no custody of funds at any point.

A customer can pay in two ways:

1. **In stablecoins.** The widget sends a plain ERC-20 `transfer` of the invoiced amount to the merchant.
2. **In the chain's native coin** (ETH on Sepolia, for example). The widget calls the stablecoin protocol's mint function with the merchant as the receiver. The protocol takes the customer's native coin into its reserve and mints the invoiced stablecoins straight to the merchant, all in the one transaction the customer signs.

The second path is the reason StablePay depends on a stablecoin *protocol* and not just on a token: automatic conversion is a protocol operation.

### The pieces

| Piece | Role |
|---|---|
| `stablepay-sdk` | React widget published to npm: `PayButton`, `Dialog`, `NetworkDropdown`, `TokenDropdown`, `TransactionReview`; plus `Config`, `NetworkSelector`, `Transaction` |
| Protocol SDK | JavaScript layer that prices a payment and builds the transaction. Was `djed-sdk`; became `tectonic-sdk` during this project |
| Merchant dashboard | Next.js app where a merchant reads payments received, filters and exports them |
| Website | Landing page, and (added in this project) the developer documentation |

### The actors

- **Merchant:** embeds the widget with a configuration (receiving address, price per network in stablecoin units).
- **Customer:** opens the widget, picks a network and token, connects a wallet and signs one transaction.
- **Stablecoin protocol contract:** holds the reserve and mints/burns stablecoins against it at an oracle price.
- **Oracle:** an on-chain price feed exposing `readData() → uint256`, the price of one stablecoin in wei of the base coin.

## Djed and Tectonic

**Djed** is the audited, production stablecoin protocol StablePay was built on. It was deployed on Sepolia, Milkomeda and Ethereum Classic. It blocks operations that would push the reserve ratio below a minimum (`reserveRatioMin`) or above a maximum, and it supports a per-integrator UI fee (`feeUI` paid to a `ui` address).

**Tectonic** is Stability Nexus's successor protocol. Its design goal is to have neither a minimum reserve-ratio threshold for minting nor the possibility to depeg. Instead of refusing operations when reserves are low, it:

- charges **stability fees** to stablecoin holders when the reserve ratio is at or below a "safe" threshold, and
- runs **triggered (forced) redemptions** before operations when the ratio falls below a "critical" threshold, redeeming selected holders' stablecoins to shrink supply and restore the ratio.

Structurally, Tectonic differs from Djed in ways that mattered for the SDK:

| Topic | Djed | Tectonic |
|---|---|---|
| Stablecoin token | Separate ERC-20 contract | **The Tectonic contract itself is the stablecoin ERC-20** |
| Reserve / equity coin | `reserveCoin()` | `equityCoin()` |
| Fixed-point scale | 1e24 | 1e18 (`D`) |
| Mint to merchant | `buyStableCoins(receiver, feeUI, ui)` | `mint(receiver)` |
| SC price for buying | `scPrice(0)` | `scPriceMint()` |
| Ratio gates | `reserveRatioMin` / `reserveRatioMax` block operations | No gates; stability fees and forced redemptions instead |
| UI fee for integrators | Yes | No |
| New, Tectonic-only reads | n/a | `balanceOfAfterStabilityFee`, `stabilityFee`, `stabilityFeeAmount` |

## What the project set out to do

The original proposal focused on improving StablePay itself. During community bonding the timeline was revised: an earlier idea of supporting multiple Djed variants was dropped in favour of studying Tectonic, because the organization had decided that StablePay should move to Tectonic.

The project ended up in three phases:

1. **Make StablePay ready for merchants (weeks 1–6).** Widget UI/UX, validation and error handling, an example site, and a run of merchant-dashboard features: pagination, sorting, progressive loading, caching, filters, CSV export and multiple addresses.
2. **Migrate from Djed to Tectonic (weeks 7–15).** Study the protocol, build the adapter layer and `tectonic-sdk`, discover that the Tectonic draft could not be deployed and used, build a patched local harness to keep moving, fix the defects upstream, take over the Tectonic workstream after the mid-term evaluation, deploy Tectonic to Sepolia, and switch StablePay to it, removing Djed entirely.
3. **Adoption and polish (weeks 13–22).** Redesign the website, write the developer documentation, record integration tutorials, gather learning resources, add Tectonic to the merchant dashboard, review other contributors' work and debug StablePay end to end.

## Where things stand at the end

- StablePay's SDK is **Tectonic-only** and is configured for the public Tectonic deployment on **Ethereum Sepolia** at `0xea262fB18992ddE398a5b6fa7A4F879FfB262aC1`.
- Tectonic on Sepolia is deployed and source-verified with a Chainlink-based inverting oracle; it started at a reserve ratio of exactly 2.0.
- The website has been redesigned and merged; the documentation site, the Sepolia switch-over, the dashboard's Tectonic option and the dashboard UI refresh are open PRs at the time of writing.
