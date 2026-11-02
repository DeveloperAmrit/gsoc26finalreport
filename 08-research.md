# 08. Research

A large share of this project was understanding before building: the Tectonic paper, the contract against the paper, how StablePay's Djed integration actually worked, and what had to change. I wrote the research up as working documents (19–25 July) so mentors and other contributors could check my reasoning. This file summarises them.

## 1. How StablePay talked to Djed

Before replacing anything I traced exactly which on-chain functions StablePay depended on. This list became the checklist Tectonic had to satisfy.

| Call | Where | Purpose |
|---|---|---|
| `stableCoin()`, `reserveCoin()` | `Transaction.init` | Get the ERC-20 addresses |
| `decimals()` on both coins | `Transaction.init` | Scaling |
| `oracle()` | `Transaction.init` | Oracle address |
| `scPrice(0)` | `tradeDataPriceBuySc` | Stablecoin price in wei |
| `fee()`, `treasuryFee()` | fee lookup | Fee rates, scaled by 1e24 |
| `buyStableCoins(receiver, feeUI, ui)` (payable) | native payment | Mint stablecoins to the merchant |
| `transfer(to, amount)` on the stablecoin | stablecoin payment | Direct payment |

Other findings from reading `djed-sdk`:

- Djed's fixed-point scale is **1e24** (`SCALING_DECIMALS = 24`).
- The cost of a native payment was `amountSC · scPrice / (1 − f − f_T − f_UI)`, including a 0.01% UI fee paid to a hard-coded beneficiary address.
- Gas was fixed at 500,000 for every transaction.
- The SDK also enforced Djed's reserve-ratio limits and a per-transaction USD limit, none of which exist in Tectonic.

## 2. The Tectonic paper

I read the June 2026 draft of the Tectonic paper, which follows the structure of the Minimal Djed paper, and mapped each equation to the Solidity.

**Assets:** base coin (the reserve asset), stablecoin (tracks an oracle target price) and equity coin (a claim on protocol equity; Djed calls it the reserve coin).

**State:** σ = (R, N_SC, N_EC, H, B_SC, T): reserve, stablecoin supply, equity-coin supply, set of stablecoin holders, balances, and each holder's last stability-fee charge time.

**Prices and accounting:**

- Mint price `P_mint = P_target` (Eq. 1).
- Redemption price `P_redeem = min(P_target, R / N_SC)` (Eq. 2). This bound is what makes a depeg below target impossible in accounting terms.
- Liabilities `L = N_SC · P_redeem ≤ R` (Eq. 3); equity `E = R − L` (Eq. 4); reserve ratio `r = R / L ≥ 1` (Eq. 5); equity-coin price `E / N_EC` (Eq. 6).

**Thresholds:** `1 < r_crit < r_safe < 2`. Below `r_crit`, triggered redemptions run before minting and after redeeming equity. At or below `r_safe`, stablecoin holders pay a stability fee.

**Operations:** mint/redeem stablecoin and mint/redeem equity coin, each charged a protocol fee `f` (stays in the reserve) and a treasury fee `f_T` (sent out).

**Stability fee (§2.9):** daily rate `((r_safe − r) / (r_safe − 1)) · f_max` when `1 ≤ r ≤ r_safe`, zero above `r_safe`; the paper's example uses `r_safe = 1.5` and `f_max = 0.000548` (about 0.0548% a day, ~20% a year). The accrued fee is paid by burning part of the holder's balance.

**Triggered redemptions (§2.10):** while `r < r_crit` and holders remain, pick a holder and redeem their entire balance. The implementation starts from a pseudo-random index for fairness, caps iterations, stops at half the remaining gas and gives the caller a small gas refund.

**Theorems (§3):** the paper lists Minimal Djed's eight properties (peg bounds, crash robustness, no insolvency, no bank runs, monotonic equity per coin, no reserve draining, bounded dilution) and leaves open which hold for Tectonic.

## 3. Assessing the contract (`e657330`, 8 July 2026)

I compared `Tectonic.sol` to the paper function by function.

| Area | Status found |
|---|---|
| Core operations (mint/redeem SC and EC) | Implemented, but blocked by the reentrancy defect |
| Pricing and accounting (Eq. 1–6) | Implemented, but `ratio()` divided by zero at zero supply |
| Stability fees (Eq. 7–8) | Implemented, but the recursion defect bricked every operation once `r ≤ r_safe` |
| Triggered redemptions (§2.10) | Implemented, with edge-case defects (empty holder set) |
| Holder tracking, events | Implemented |
| Oracle | `IOracle` + `ChainlinkOracle` |
| Speculation fee, staking interest, tx limits at low ratio, volume-based spreads | Not implemented (TODOs; volume counters declared but never written; `speculationPeriod = 10 * 1e16` commented as "10%", a placeholder since a period should be a duration) |
| Tests, deploy scripts, usable deployments, audits | None |

Other observations: the Tectonic contract **is** the stablecoin ERC-20 (Djed deploys a separate token); fees scale by **1e18** not 1e24; `ecPrice()` was `internal`; the stablecoin's name and symbol were hard-coded as `"StableCoin"`/`"SC"`; the equity coin's symbol was still `"RC"` from the reserve-coin naming; and the constructor validated none of the paper's parameter constraints.

## 4. The integration plan: Djed to Tectonic

I mapped every StablePay need to Tectonic and marked what existed and what was missing.

| StablePay need | Djed | Tectonic | Status at the time |
|---|---|---|---|
| Stablecoin token | separate ERC-20 | same address as Tectonic | Have (SDK change) |
| Equity coin | `reserveCoin()` | `equityCoin()` | Have |
| Oracle | `oracle()` | `oracle()` | Have |
| Price for buying | `scPrice(0)` | `scPriceMint()` | Have |
| Price for selling | `scPrice(0)` | `scPriceRedeem()` | Have |
| Fees | `fee()`, `treasuryFee()` @1e24 | `fee()`, `treasuryFee()` @1e18 | Have (rescale) |
| Mint to merchant | `buyStableCoins(receiver, feeUI, ui)` | `mint(receiver)` | Have, but blocked by a defect; no UI fee |
| Direct payment | `transfer` | `transfer` | Have, but with stability-fee side effects |
| Ratio-gate checks | `reserveRatioMin/Max`, tx limit | not applicable | Remove from SDK |
| Equity-coin price | `rcBuyingPrice(0)` | `ecPrice()` internal | Missing (requested public) |
| UI-fee revenue | `feeUI` | none | Missing (product decision) |
| Effective balance | n/a | `balanceOfAfterStabilityFee` | New, SDK must use it |
| Deployed addresses | 3 networks | none | **Hard blocker** |
| ABI artifact | `DjedABI.json` | none | Build from Foundry output |

**New cost formula.** Djed: `amountSC · scPrice / (1 − f − f_T − f_UI)` at 1e24. Tectonic: `amountSC · scPriceMint / (1 − fee/D − treasuryFee/D)` at 1e18, **rounded up** because `mint()` floors its own division, so a quote rounded down would underpay the merchant.

**Behaviours the widget had never had to handle:**

1. Merchant balances can decay through stability fees when `r ≤ r_safe`.
2. A merchant's stablecoins can be force-redeemed into native coin when `r < r_crit`.
3. `mint()` gas varies with the number of forced redemptions, so gas must be estimated.
4. A plain stablecoin transfer charges stability fees to both parties and costs more gas than a normal ERC-20 transfer.

This plan drove the adapter design in [04](04-stablepay-sdk.md).

## 5. Questions for the Tectonic maintainers (25 July)

I wrote 23 questions, tiered by how much each blocked StablePay, each with the observation, why it mattered and a proposed answer where one was obvious, plus a one-paragraph version for a five-minute read.

- **Tier 1 (blocking):** the stability-fee recursion; `ratio()` at zero supply; the nested reentrancy guard; the empty-holder panic; the fee window opening at the Unix epoch for a new holder. Each had a test that failed upstream and passed on my patched copy.
- **Tier 2 (paper vs. code):** fees on every transfer vs. only on mint/redeem; `f_max` as a constant vs. a parameter; missing constructor validation; small paper corrections (Eq. 8's "8640000 milliseconds is a day" should be 86,400,000; §2.6 has the `< 2` bound on `r_safe` but §2.11 omits it).
- **Tier 3 (interface stability):** is `mint(address)` final; will there be a UI fee; can `ecPrice()` be public; will name, symbol and decimals become constructor parameters; will there be a factory or registry.
- **Tier 4 (economics and safety):** a mint can pass the critical check and still leave `r < r_crit`; which Djed theorems hold; whether permissionless `forceRedemptions` with a `tx.origin` refund and a caller-influenced starting index is open to griefing or MEV; what merchants should be told about forced redemption; which TODOs are in scope for v1.
- **Tier 5 (deployment and oracles):** StablePay's live chains (ETC, Milkomeda) have no Chainlink feeds while Djed ships nine oracle adapters for exactly that reason; which chain first; the audit plan.
- **Tier 6 (process):** whether to send PRs for the Tier 1 fixes, whether a test-suite contribution was welcome, where to discuss.

Answers to these shaped the rest of the project: the Tier 1 fixes became Tectonic #22 and #24; the supply/equity discussion led to the seeded supplies and the super-critical floor; the interface and fee-recipient discussion led to #26; and the oracle and network questions decided the Sepolia deployment with a Chainlink inverting oracle.

## 6. Protocol observations

These came from running the contract, not just reading it.

### A mint can leave the ratio below critical

`mint()` checks the ratio after `msg.value` has already been credited to the contract balance but before the new coins exist. Both push the measured ratio up compared with the state that actually results. On my local deployment: the protocol sat at r = 1.10 with 4.975 ETH reserve, critical at 1.20. A 5 ETH mint measured the ratio at about 2.20, skipped redemptions, issued coins at the target price and settled at **r ≈ 1.056, below the critical threshold it had just passed**. I read this as consistent with Tectonic having no minimum ratio for minting, but it means "after a mint, r ≥ r_crit" is not an invariant. For StablePay that is concrete: a merchant can be paid and then force-redeemed by the next transaction. A test pins the behaviour.

### Stability fees on transfers

Paper §2.9 says fees are charged "only ... before minting and redeeming SCs". The implementation charges them from `_update`, so on every transfer, to both sender and receiver. StablePay's stablecoin payment path is a plain transfer, so this changes both the cost and what the merchant receives.

### Recovery depends on the protocol fee

The argument that forced redemptions quickly restore `R > L` relies on the fee retained on each redemption. With `fee == 0`, redemptions leave reserve-per-coin unchanged and the ratio never improves (see [05](05-tectonic-contracts.md)).

### The seed and the stability-fee threshold

With the 0.01 ETH Sepolia seed, minting about 0.005 ETH of stablecoins takes the ratio below 1.5 (where stability fees start), and about 0.045 ETH takes it below 1.1. This is why adding equity to the Sepolia deployment is on the follow-up list.

## 7. Oracle research

- Tectonic's price convention is "wei per stablecoin", the inverse of what Chainlink's ETH/USD feed reports. Using the existing `ChainlinkOracle` on that feed would have priced 1 SC at about 2,236 ETH. This led to `ChainlinkInvertingOracle`.
- I studied Djed-Solidity's oracle family (API3, HebeSwap, Chainlink, inverting variants, the Shu converter), which exists because Ethereum Classic and Milkomeda lack Chainlink feeds. The `IOracle` interface is the same (`readData() → uint256`), so those adapters could be ported for mainnet.
- I considered OrbOracle for the first testnet and decided against it: it needs active price submitters, a `readData()` adapter and a governance token. My community-bonding work on the OrbOracle poster (see [10](10-community-bonding-and-preparation.md)) gave me first-hand knowledge of that operational cost.

## 8. Infrastructure research

- **Archive logs.** publicnode prunes `eth_getLogs` history, which broke the dashboard on the Djed and Tectonic Sepolia ranges. Tenderly's public gateway keeps full history and allows browser requests, so the dashboard reads through it with publicnode as fallback.
- **Default RPCs.** viem's default Sepolia RPC rejects free requests, so the SDK pins the publicnode endpoint.
- **Sepolia gas rules.** Sepolia activated the Glamsterdam upgrade on 6 October 2026; EIP-8037 charges state gas per byte of deployed code, which my Foundry version didn't model yet. Details in [09](09-tinkering-and-experiments.md#6-debugging-the-sepolia-deployment).
