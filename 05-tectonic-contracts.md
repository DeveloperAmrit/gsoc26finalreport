# 05. Tectonic Contracts

[StabilityNexus/Tectonic-EVM-Contracts](https://github.com/StabilityNexus/Tectonic-EVM-Contracts) is the Solidity implementation of the Tectonic stablecoin protocol (Foundry, solc 0.8.23, OpenZeppelin). This was not part of my original proposal. I started reading it because StablePay had to integrate with it, found that it could not be used as it stood, fixed it upstream, and after the mid-term evaluation took over the Tectonic workstream, ending with a public testnet deployment.

My mentor on the contract side was Bruno, who reviewed and merged every PR here. Most of the design discussion happened on Discord.

## 1. The state of the contract when I started

I assessed the contract at commit `e657330` (8 July 2026). Summary of what I found (the full assessment is in [08-research.md](08-research.md)):

- Core operations were implemented: `mint`, `redeem`, `mintEquityCoins`, `redeemEquityCoins`, `forceRedemptions`, stability fees through the ERC-20 `_update` hook, holder tracking, a Chainlink oracle adapter.
- There were **no tests** beyond the Foundry template's `Counter.t.sol`, **no deploy script**, and **no deployments** that could actually be used.
- Several protocol features were TODOs: speculation fee, staking interest, transaction limits at low ratio, volume-based spreads.
- And the contract had defects that made it **unusable**: the first `mint()` on a fresh deployment reverted, `redeem()` always reverted, and once the reserve ratio dropped to the safe threshold, every transfer, mint and redeem ran out of gas.

I found each defect while trying to integrate StablePay against a local deployment. I reproduced each one with a failing test in a local copy of the contract before proposing a fix (see [09](09-tinkering-and-experiments.md)).

---

## 2. PR #22: three defects that made the contract unusable (`fix-2-5-8`, merged Aug 11)

### 2.1 Nested reentrancy guard

`_redeem` was `internal nonReentrant`, but its only callers, `redeem` and `forceRedemptions`, were themselves `nonReentrant`. OpenZeppelin's guard reverts when the lock is already held, so **`redeem()` could never succeed**. The same applied to `mint`, `mintEquityCoins` and `redeemEquityCoins`, each of which calls `forceRedemptions` internally. Any operation that tripped the critical reserve ratio reverted instead of redeeming.

**Fix:** removed the modifier from the internal `_redeem`, and split `forceRedemptions` into a guarded external entry point plus an unguarded internal `_forceRedemptions` for callers that already hold the lock. Every externally reachable function still carries `nonReentrant`, so protection is unchanged in effect; the guard just stops rejecting the contract's own calls.

### 2.2 Unbounded recursion in `chargeStabilityFee`

This was the most serious defect and the hardest to see by reading the code.

```solidity
function chargeStabilityFee(address a) public returns (uint256) {
    uint256 feeAmount = stabilityFeeAmount(a);
    if (feeAmount > 0) _burn(a, feeAmount);   // interaction
    timestamp[a] = block.timestamp;           // effect, after
    ...
}
```

The contract charges a stability fee from `_update`, which runs before every transfer, mint and burn. So `_burn` calls back into `chargeStabilityFee`. Because `timestamp[a]` hasn't been written yet, the fee recomputes as non-zero and burns again. The balance decays geometrically rather than reaching zero, so the recursion only ends when the transaction runs out of gas. I measured **about 1.06 billion gas** for one call before the revert.

The trigger condition is not exotic: any operation once the reserve ratio is at or below `safeReserveRatio` and any time has passed. In that state **every transfer, mint and redeem reverted**. The mechanism meant to stabilise the protocol under stress was the thing that disabled it under stress.

**Fix:** write `timestamp[a]` before the burn (checks-effects-interactions). The re-entrant call sees zero elapsed time and returns immediately. No fee is lost, because the amount is read before the timestamp moves. Cost dropped from ~1.06 billion gas (then revert) to under 30,000.

### 2.3 `ecPrice()` was `internal`

No off-chain caller could read the equity coin's price. I made it `public`. This discloses nothing new, since the value was derivable from `E()` and the equity coin's supply, both already public.

### Scoping

I deliberately kept this PR to three changes in one file (+22 / −6) so it would be easy to review. In the PR description I listed the other issues I had found (zero-supply division, empty holder set, epoch-based fee window, missing constructor validation, divide-by-zero views) and two design questions, and offered follow-up PRs.

---

## 3. PR #24: seed both supplies at deployment (`fix-1-fresh`, merged Aug 31)

### The problem

`ratio()` computes `D * R() / L()`, and `L()` is zero whenever there are no stablecoins, which is the state of every contract the moment it is deployed. Both `mint()` and `mintEquityCoins()` read `ratio()` first, so **a fresh deployment reverted on every entry point and could never be bootstrapped**.

### The approach

My first local fix returned `type(uint256).max` from `ratio()` when `L() == 0`. In review, Bruno suggested making zero denominators structurally impossible instead. I implemented that:

- The constructor is now `payable` and **requires a non-zero payment**.
- The payment backs an initial stablecoin supply and equity-coin supply that are **owned by no address**. Nobody can transfer, burn or redeem coins that nobody owns, so the reserve and both supplies stay permanently positive, and `ratio()` and `cushion()` are well defined without any per-call checks. This is the same idea as Uniswap V2's `MINIMUM_LIQUIDITY`.

### Two decisions I had to make

**How to hold the seed.** "Increase total supply without giving the coins to anyone" isn't possible through OpenZeppelin's ERC-20 API: `_totalSupply` is `private`, `_mint(address(0), x)` reverts with `ERC20InvalidReceiver`, and `_update(address(0), address(0), x)` adds and subtracts for a net of zero. So the seed lives in an `immutable` (`seedSupply`) and is added in a `totalSupply()` override, in both `Tectonic` and `Coin`. The sum of balances is then less than `totalSupply()` by exactly the seed. That is intended, but I flagged that some indexers and explorers would notice it. The alternative, minting to a burn address, would have made the seed a *holder* that forced redemptions could redeem, destroying the guarantee unless that address were special-cased.

**How to split the payment.** Evenly. Backing all of it with stablecoins would put `L == R` and leave equity at zero from block one, the very state this change exists to avoid. An even split starts the protocol at a **reserve ratio of exactly 2** with `ecPrice() == 1`.

### A guard that belongs with it

The seed makes "positive supply, no holders" the *initial* state of every deployment. If the oracle price rose enough to push the ratio below critical before anyone minted, the next `mint()` would enter the redemption loop and compute `% (holders.length - 1)`, that is `% 0`, and panic. So I added an early return for an empty holder set in `_forceRedemptions`.

### Other notes from the PR

- The constructor now reads the oracle, so a broken or zero oracle fails at deploy time instead of producing a silently unusable contract.
- `seedSupply` is immutable, so there is no storage layout change.
- This is a breaking deployment change: deploy scripts must attach value. I asked whether the Sepolia and Polygon Amoy rows already in `Deployments.md` were in use, since those instances could not adopt the new constructor without redeploying.
- I checked with `git merge-tree` that `fix-1-fresh` and `fix-4-6-fresh` conflicted with neither each other nor `main`.

---

## 4. PR #25: zero equity, then a super-critical floor (`fix-4-6-fresh`, merged Oct 18)

### First version: handle `R == L`

Seeding guarantees non-zero *supplies*, but not non-zero *equity*. `E() = R() − L()` reaches zero through ordinary market movement: once the target price can no longer be met, `scPriceRedeem()` clamped to `R / supply`, which made `L` exactly `R` and `E` exactly `0`. Three places then panicked:

| Function | At `E == 0` before | After (first version) |
|---|---|---|
| `leverage()` | divide by `E()`, panic | `type(uint256).max` (leverage is unbounded) |
| `yieldFromStabilityFeeDaily()` | divide by `ecPrice() == 0`, panic | `0` (no equity to earn on) |
| `mintEquityCoins()` | divide by `ecPrice()`, panic | revert with a reason |

I flagged the third as a design question: if equity is zero, the protocol can't accept new equity in exactly the state where it most needs recapitalising. Minimal Djed handled this with a floor on the reserve-coin price (`rcMinPrice`).

I also pointed out an edge case in the argument that forced redemptions restore `R > L` quickly. That recovery depends on the protocol fee: redeeming `x` stablecoins removes `x·p·(1−f)` from the reserve but `x` from supply, so the ratio only improves because of the fee left behind. **With `fee == 0`, a redemption leaves reserve-per-coin unchanged and `R == L` would persist forever.** I suggested requiring a non-zero fee.

### Second version: `superCriticalReserveRatio`

Following the review discussion, the guards were replaced with a change to the pricing itself. I added a new immutable parameter, the **super-critical reserve ratio** `r_supcrit`, and changed the redemption price:

```
Let q = R / (N_SC · P_target).

P_redeem = P_target                    if q ≥ r_supcrit
P_redeem = (1 / r_supcrit) · R / N_SC  if q < r_supcrit
```

Below the threshold, `L = R / r_supcrit`, so **the reserve ratio is pinned at `r_supcrit` instead of falling to 1**, and

```
E ≥ ((r_supcrit − 1) / r_supcrit) · R > 0   whenever R > 0.
```

Equity, and so `ecPrice()`, can never reach zero, which made the zero-equity guards unnecessary. I also:

- updated `stabilityFee()` so the fee approaches its maximum as the ratio approaches `r_supcrit` (its new floor) instead of 1;
- added constructor validation `1 < superCritical < critical < safe`, which the paper requires but the contract had never checked;
- documented the derivation as comments next to `scPriceRedeem()` and `ecPrice()`;
- updated the constructor-parameter columns in `Deployments.md`.

### CI and static analysis

- The gas-snapshot workflow failed on every PR because `ChainlinkOracle.sol` imports `@chainlink/contracts` through a `node_modules` remapping, and the job never ran `npm install`. I added the step.
- Went through the Slither findings: rejected a zero treasury address in the constructor (fees sent there would be burned), reordered the seed computation to multiply before dividing, and annotated the three findings that are intentional (unused return values from `latestRoundData`, the predictable starting index for forced redemptions, and a strict equality used only as a shortcut), each with the reason.

---

## 5. PR #26: dev, dev-fee receiver and dev fee (`dev-fee-receiver`, merged Oct 23)

The organization wanted the fee recipient to be manageable after deployment. I replaced the immutable `treasury` / `treasuryFee` pair with:

- a **`dev`** address set in the constructor;
- a **`devFeeReceiver`** that receives `devFee` on every mint and redeem, changeable by `dev` through `setDevFeeReceiver`;
- a **mutable `devFee`**, changeable by `dev` through `setDevFee`, emitting `DevFeeUpdated`;
- a **two-step dev handover**: `proposeDevTransfer(newDev)` by the current dev, then `acceptDevTransfer()` by the proposed address, so a typo can't hand the role to an address nobody controls (proposing `address(0)` cancels);
- `require(fee + devFee < 100%)` in both the constructor and `setDevFee`, so `deductFees` can never underflow and block mints and redeems;
- zero-address checks for `dev` and `devFeeReceiver`.

---

## 6. Oracle work for the deployment

`scPriceMint()` returns `oracle.readData()` directly and treats it as **the price of one stablecoin in wei**. The Chainlink ETH/USD feed reports the opposite direction (USD per ETH, 8 decimals). The existing `ChainlinkOracle` only rescaled decimals, so on that feed it would have priced one stablecoin at roughly 2,236 ETH.

I wrote `ChainlinkInvertingOracle`, which returns `10^(feedDecimals + 18) / answer`, that is wei per USD. It follows the inverting oracle in Djed-Solidity, with stricter checks. It reverts on:

- a non-positive answer;
- an incomplete round (`startedAt == 0` or `updatedAt == 0`);
- a stale round (`answeredInRound < roundId`);
- an answer older than `maxStaleness` (set to one day for Sepolia).

**Why Chainlink and not OrbOracle.** I considered OrbOracle, Stability Nexus's own oracle. For a first testnet it would have needed someone to keep submitting prices, an adapter to the `readData()` interface and a governance token, which was more moving parts than the deployment needed. We went with Chainlink for Sepolia; mainnet oracles are an open item (Ethereum Classic has no Chainlink feed).

---

## 7. Deployment to Ethereum Sepolia (PR #27)

### What I added to the repository

| Commit | Change |
|---|---|
| `41d2180` | `src/ChainlinkInvertingOracle.sol`, `script/DeployTectonic.s.sol`, `test/ChainlinkInvertingOracle.t.sol`, `test/DeployTectonic.t.sol`, `test/MockAggregator.sol`; switched the `sepolia` RPC alias from `rpc.sepolia.org` to the publicnode endpoint |
| `92b1176` | Lowered the deployment seed from 0.05 to 0.01 ETH |
| `dc59cec` | Recorded the deployment in `Deployments.md` and marked the older Sepolia and Polygon Amoy rows as deprecated |

No logic in `Tectonic.sol` was changed for the deployment.

**`DeployTectonic.s.sol`** deploys the oracle and then Tectonic with the seed in a single run, refuses to run on any chain other than Sepolia (chain ID 11155111), takes `dev` from the broadcasting account, and logs every address and parameter with the initial price and reserve ratio.

### Tests

All 13 tests pass on Foundry 1.5.1 / solc 0.8.23, and the new files pass `forge fmt --check`.

- **`ChainlinkInvertingOracle.t.sol` (8 tests):** inverted price, scaling across feed decimals, staleness at and past the limit, stale round, non-positive answers, constructor input checks.
- **`DeployTectonic.t.sol` (3 tests):** runs the real deploy script against a mock feed placed at the Chainlink address, checks every parameter and a ratio of 2.0, performs a mint and a full redeem, and checks that it refuses other chains.

Before broadcasting I also:

- **rehearsed on a local fork of Sepolia** reading the live Chainlink feed, and ran `mint`, `redeem` and `mintEquityCoins` successfully;
- **dry-ran against Sepolia** from the deployer wallet and checked the simulated parameters and a ratio of 2.0.

### The deployment (30 October 2026, block 12,027,502)

| Contract | Address |
|---|---|
| Tectonic (StableCoin, `SC`) | `0xea262fB18992ddE398a5b6fa7A4F879FfB262aC1` |
| EquityCoin (`RC`) | `0x22B2f0CAf598B9E5FCf888fB89eD50c2F4735F1a` (created inside the Tectonic constructor) |
| ChainlinkInvertingOracle | `0x4f007748666F2b941AA014E5a2a41841e681A32b` |

All three are source-verified on Etherscan. Total cost was 0.000157 ETH in gas (23,280,684 gas at about 0.0067 gwei), plus the 0.01 ETH seed now in the reserve.

**Parameters** (fixed-point base `D = 1e18`):

| Parameter | Value | Reason |
|---|---|---|
| `fee` | 0.1% | Stays in the reserve; same as the earlier Sepolia row |
| `devFee` | 0.1% | Sent to `devFeeReceiver` on each mint and redeem |
| `superCriticalReserveRatio` | 1.05 | New parameter; must sit between 1 and critical, so the midpoint |
| `criticalReserveRatio` | 1.1 | Below this, triggered redemptions run |
| `safeReserveRatio` | 1.5 | At or below this, stability fees accrue; matches the paper's example |
| `dev` | deployer wallet | Keeps fee changes possible during testing; can be handed over in two steps |
| `devFeeReceiver` | `0x593a…CE65` | Tectonic treasury address provided by Bruno |
| Seed | 0.01 ETH | Limited by available Sepolia ETH; starts the protocol at ratio 2.0 |

**State read back from the chain right after deployment:** reserve 0.01 ETH, ratio 2.0, `scPriceMint()` = 447,159,197,617,535 wei per SC (ETH ≈ $2,236), stablecoin seed supply 11.1817 SC, equity-coin supply 0.005 RC.

### Why Sepolia

It has a Chainlink ETH/USD feed, it was the only testnet already listed in `Deployments.md`, Djed had been deployed there, and StablePay already had a Sepolia network entry. Mordor (Ethereum Classic's testnet) has no Chainlink feed. Base Sepolia can follow later as a rehearsal for Base mainnet.

The problems I hit while deploying (a gas-rule change on Sepolia, network resets, faucets) are written up in [09-tinkering-and-experiments.md](09-tinkering-and-experiments.md#6-debugging-the-sepolia-deployment).

---

## Open findings

These came out of my test suites and are not fixed upstream yet. Three are pinned by `UPSTREAM_BUG` tests in StablePay's synced harness (#109):

1. **The forced-redemption sweep reverts when a stability fee is owed.** `_redeem` and the sweep size the burn from the pre-fee balance, but `_update` burns the fee first, so the outer burn fails with `ERC20InsufficientBalance`. Below the critical ratio this is almost always the case after any elapsed time. My local fix: charge the fee before pricing the redemption and before reading the holder's balance.
2. **A holder that rejects base coin blocks the sweep.** A holder whose `receive()` reverts or burns gas makes every sweep revert. My local fix: pay forced redemptions with a gas-capped call that copies no return data, credit failed payouts to an `unclaimedBC` balance withdrawable through `claimUnpaidRedemption()`, and exclude credited coin from `R()`.
3. **The sweep panics once holders run out below critical.**

Design questions I raised that are still open:

- The paper says stability fees are charged only on mint and redeem; the contract charges them on every transfer, to both parties. This matters to StablePay because its direct-payment path is a transfer.
- A mint can pass the critical-ratio check and still leave the ratio below critical, because `msg.value` is credited before the check and the coins are issued after it.
- `f_max` and the redemption-iteration cap are constants, not deployment parameters.
- `forceRedemptions` is permissionless, refunds gas to `tx.origin` from the reserve, and picks its starting holder from `keccak256(block.timestamp, msg.sender)`, which a caller can influence.
