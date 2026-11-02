# 04. StablePay SDK and Widgets

This file covers everything I did in [DjedAlliance/StablePay](https://github.com/DjedAlliance/StablePay): the widget, the core payment logic, the protocol adapter, the new `tectonic-sdk` package, the migration to Tectonic and the final switch to the public Sepolia deployment.

## 1. Where the SDK started

At the start of the project, `stablepay-sdk` was a React widget backed by `djed-sdk`, a web3.js library for Djed. The flow lived mostly in `TransactionReview.jsx`:

1. Create a `Transaction` for the chosen network's Djed address and `init()` it (read the Djed, stablecoin, reserve-coin and oracle contracts).
2. For a native-coin payment, call `handleTradeDataBuySc(amount)` to compute how much native coin the customer must send (price plus protocol, treasury and UI fees).
3. Connect the wallet, switch it to the right chain, then either:
   - **native path:** `buyStableCoins(merchant, feeUI, ui)` with the computed value, so Djed mints stablecoins to the merchant; or
   - **stablecoin path:** a plain ERC-20 `transfer(merchant, amount)`.
4. Send with viem, wait for the receipt, show the explorer link, fire the completion callback.

Three networks were configured (Sepolia, Milkomeda C1, Ethereum Classic), each with a Djed address and a separate stablecoin token address. Gas was hard-coded at 500,000, and a UI-fee beneficiary address was hard-coded in two places.

## 2. Phase 1: making the widget ready for merchants (PR #105)

### An example website that tracks local builds

The existing demo repository installed `stablepay-sdk` from npm, so the only way to see a widget change there was to publish a new version. I added `stablepay-sdk/example/`, a React + Vite app that imports the SDK from the local build. Every `npm run build` in the SDK shows up in the example immediately. This sped up all later work: every widget change, the Tectonic adapter and the final Sepolia switch were developed against it.

### UI/UX

- Restyled `PricingCard.css` and refactored `Dialog.jsx` and `TransactionReview.jsx` for a cleaner layout and better responsiveness on small screens.
- Moved the widgets from a dark to a **bright theme**, replaced the logo, and removed a confirm step that added a click without adding information.

### Validation and error handling (issues #60, #61, #62)

- Reworked the payment flow around explicit interaction states so the widget always knows whether it is idle, quoting, waiting for the wallet, waiting for confirmation, finished or failed, and renders exactly one of those.
- **No wallet installed:** the widget used to freeze. It now explains that a wallet is needed instead of hanging.
- **Wallet installed but logged out / locked:** handled separately with its own message.
- Clearer messages for wrong network, connection/DNS failures and reverted transactions.
- Better input validation in `Transaction.js` before anything is sent.

## 3. Phase 2: Tectonic behind a protocol adapter (PR #106)

When I began the Tectonic integration there was no public Tectonic deployment, and the upstream draft could not be used even locally (see [05](05-tectonic-contracts.md) and [09](09-tinkering-and-experiments.md)). I designed the integration so it could be built and tested against a local chain, without breaking the working Djed paths.

### The adapter layer

- A `ProtocolAdapter` interface describing every chain interaction StablePay performs: initialize, quote a native payment, build the mint transaction, build the direct transfer, get the stablecoin address and decimals, get warnings.
- `DjedAdapter` wrapped the existing `djed-sdk` calls. `TectonicAdapter` wraps the new `tectonic-sdk`.
- `Transaction.js` became a thin delegator that keeps its old public method names (`init`, `handleTradeDataBuySc`, `buyStablecoins`), so the widget components needed very few changes.
- Each network entry in `utils/config.js` gained a `protocol` field, defaulting to `"djed"`, so every existing network behaved exactly as before.

### `tectonic-sdk`, a new package

- **`pricing.js`**: pure functions with no network access, mirroring `Tectonic.sol` exactly:
  - `netFactor(fees)`: `D − fee − devFee`, throwing if fees would consume the whole payment.
  - `stablecoinsForPayment(amountBC, price, fees)`: reproduces the contract's two separate floor divisions rather than folding them into one, because the results can differ by 1 wei and the merchant guarantee depends on matching the contract exactly.
  - `requiredPaymentForStablecoins(amountSC, price, fees)`: the inverse, **rounded up** with a 1-wei safety margin so the merchant is never credited less than the invoice.
  - `payoutForRedemption`, `totalFeePercent`, `ceilDiv`.
  - `toBaseUnits` / `fromBaseUnits`: decimal conversion without floating point.
- **`tectonic.js`**: a viem-based `TectonicClient` with `init`, `scPriceMint`, `scPriceRedeem`, `getState`, `getReserveHealth`, `getBalance` (raw and after stability fee), `quoteMint`, `quotePayment`, `quoteRedeem`, `buildMintTx`, `buildTransferTx`, `buildRedeemTx`.
- **`artifacts/TectonicABI.js`**: the ABI as a plain JS export rather than JSON, because importing JSON from ESM needs import attributes whose syntax differs across Node versions and a Rollup plugin; a JS module works everywhere with no build config.

### Design decisions that came out of the protocol differences

| Decision | Why |
|---|---|
| The Tectonic address is also the stablecoin address | Tectonic *is* the ERC-20. `getStablecoinAddress()` returns the protocol address, and the direct-transfer path targets it. No second address in config. |
| Fees validated against a 1e18 scale | Djed scales by 1e24. A Djed-scaled 1.5% fee (15e21) is a million times too large for Tectonic. `netFactor()` rejects any fee ≥ `D`, so a porting mistake fails loudly instead of silently mispricing every invoice. A regression test pins this. |
| Quotes round up and are refreshed right before signing | The oracle price moves between opening the dialog and signing. The widget re-quotes immediately before building the transaction. |
| Gas is estimated, not hard-coded | Tectonic's `mint()` may run up to 100 forced redemptions first; the cost is only known at execution time. A fixed 500k limit would strand transactions exactly when the protocol is stressed. The SDK estimates and applies a 1.5× margin, falling back to wallet estimation if the estimate fails. |
| Merchants are warned about Tectonic-only behaviour | `getReserveHealth()` classifies the protocol as *healthy*, *fee-accruing* or *critical*. The widget shows the daily stability-fee rate when fees are accruing, and warns that balances may be force-redeemed when the ratio is critical. A failed diagnostic read never blocks a payment. |
| Effective balances | A raw `balanceOf` overstates holdings when fees are accruing. `getBalance()` returns both the raw and the `balanceOfAfterStabilityFee` value. |

### A display bug fixed on the way

The success callback and the pay button reported the stablecoin invoice amount with the payment token's symbol, so a 0.00255 ETH payment for a 5 SC invoice showed as "5 ETH". The receipt now distinguishes the amount paid from the amount received.

### Testing

- 21 Solidity tests in the local harness, including two fuzz suites at 512 runs.
- 28 SDK unit tests with no chain required, including a 192-case sweep proving the merchant is never short-changed across amounts, prices and fee sets.
- 14 end-to-end checks against a live local deployment (`npm run smoke`).
- Both payment paths driven by hand through the widget on Anvil.

Atharva Naik, my pair-programming peer, reviewed the PR. I fixed the issues he found in the pricing code and the review screen before it was merged.

## 4. Mid-term work (PR #108)

At the mid-term evaluation, Bruno recommended bringing the repository up to the organization's standard. I added:

- `README.md` rewritten, `CONTRIBUTING.md`, `SECURITY.md` (supported versions per package), `MAINTAINERS.md`, `LICENSE`.
- A best-practices checklist with a JSON status file, and an OpenSSF Scorecard workflow.
- A complete `brand/` folder: `Brand.md`, palette in CSS/JSON/SVG, typography, colour and mono logos, the full favicon set and web manifest, and `generate-rasters.py`, which produces every PNG/ICO size from the SVG source. The script has its own unit tests that check names, sizes, colour modes and ICO frames.
- The example app picked up the new favicons and manifest.

I later added hardening commits to the same branch:

- **Exact Djed quote.** The Djed path built the payment from a 6-decimal display string, which could underpay by up to 1e12 wei. It now uses the unscaled value.
- **Stale-write guard.** If the user changes network or token while a quote is loading, the older run can no longer overwrite the newer quote, warnings or details.
- **Wallet error codes.** 4001 (user rejected) and −32005 (rate limited) are no longer shown as connection failures.
- **Dev-only local network.** Hidden from merchants unless explicitly enabled.
- **Packaging.** `djed-sdk` external at the published version, `tectonic-sdk` bundled, and a narrower Vite `fs.allow`.
- **Fee-scale validation** in every pricing function, not just one.

## 5. Removing Djed (PRs #107 and #110)

The organization decided StablePay should support Tectonic only. #107 removed all three Djed integrations. It went stale while #106 was in review, so I re-applied the removal on current `main` as #110:

- Deleted `djed-sdk/` (about 16,000 lines including artifacts and lock files), `DjedAdapter`, `ProtocolAdapter` and the adapter factory. `Transaction` constructs a `TectonicAdapter` directly and rejects the old `(uri, address)` constructor form with a clear error message.
- `utils/config.js` lists only Tectonic deployments.
- Dropped the `djed-sdk` and `web3` dependencies and their Rollup externals.
- The example app shows setup steps when no Tectonic address is configured.
- README, comments and `.gitignore` cleaned of Djed references.

## 6. Keeping the local harness honest (PR #109)

Once my upstream fixes were merged, I synced the local copy so StablePay was testing the real contract rather than my patched one:

- `Tectonic.sol`, `Coin.sol`, `IOracle.sol` and `Math.sol` became byte-identical to upstream. Only `MockOracle.sol` stayed local.
- Picked up the super-critical reserve ratio and the payable constructor that seeds supplies.
- Rewrote the test suite for the synced contract (32 tests). Three tests marked `UPSTREAM_BUG` pin defects still present upstream (see [05](05-tectonic-contracts.md#open-findings)).
- `tectonic-sdk` exposes `superCriticalReserveRatio` and `seedSupply`.
- `toBaseUnits` now rejects a JS number unless it is a safe integer. `0.1 + 0.2` is `0.30000000000000004`, so a fractional number may already be wrong before it reaches the SDK; amounts must be passed as decimal strings or bigints.
- **Doomed-mint guard.** Below the critical ratio, the synced contract reverts during forced redemptions if any holder owes a stability fee. If gas estimation fails in that state, `buildMintTx` throws a clear error instead of returning a transaction the customer would sign and lose gas on.

## 7. Going live on Sepolia (PR #111)

After deploying Tectonic to Sepolia myself (see [05](05-tectonic-contracts.md#7-deployment-to-ethereum-sepolia-pr-27)):

- **ABI match.** The deployed contract uses `dev`, `devFeeReceiver` and `devFee` instead of `treasury` and `treasuryFee`, and the dev can change `devFee`. I renamed the fee field throughout pricing and the client, and the SDK now re-reads `devFee` on every quote, so a fee change can never leave a merchant short.
- **Network config.** Sepolia is StablePay's network, at `0xea262fB18992ddE398a5b6fa7A4F879FfB262aC1`, with the publicnode RPC because viem's default Sepolia RPC rejects free requests. The local network, the localhost chain and `useLocalTectonic()` are gone.
- **Smoke test.** `smoke-sepolia.mjs` checks the live deployment read-only by default, with an optional real mint.
- **`tectonic-local` removed** (about 1,100 lines).
- **Mint verification.** `mint(receiver)` has no minimum-output parameter. If the oracle price or `devFee` rises between the quote and the mint, the transaction still succeeds but credits the merchant less than the invoice. The widget now decodes the `Minted` event from the receipt (`verifyMint`) and calls `onTransactionComplete` only when the minted amount covers the invoice. Otherwise it shows an underpayment message and calls a new optional `onTransactionUnderpaid(invoiced, received, shortfall)` callback, so the merchant's site can handle it.

## 8. The SDK at the end of the project

```
StablePay/
├── stablepay-sdk/
│   ├── example/                  React + Vite app, pays into Sepolia
│   └── src/
│       ├── core/
│       │   ├── Transaction.js     delegates to TectonicAdapter
│       │   └── adapters/TectonicAdapter.js
│       ├── utils/config.js        Sepolia Tectonic deployment
│       └── widget/                PayButton, Dialog, dropdowns, TransactionReview
└── tectonic-sdk/
    ├── src/pricing.js             contract-exact pricing math
    ├── src/tectonic.js            TectonicClient (viem)
    ├── src/artifacts/TectonicABI.js
    ├── scripts/smoke-sepolia.mjs
    └── test/                      pricing and client tests
```

A merchant integration looks like this:

```jsx
import StablePay from 'stablepay-sdk';

const config = new StablePay.Config({
  receivingAddress: '0xMerchant...',
  amounts: { sepolia: { stablecoin: 10 } },
});
const networkSelector = new StablePay.NetworkSelector(config);

<StablePay.Widget
  networkSelector={networkSelector}
  onTransactionComplete={onPaid}
  onTransactionUnderpaid={onShort}
/>
```
