# 09. Tinkering and Experiments

This file covers the work that doesn't show up as a feature: the local Tectonic harness I built to unblock the migration, the test suites, the experiments that pushed the protocol into stressed states, the end-to-end smoke tests, and the debugging behind the Sepolia deployment.

## 1. Why a local harness

In July, StablePay needed a Tectonic contract to integrate against, and none existed that worked:

- there was no public deployment;
- the upstream contract (`e657330`) compiled and deployed, but **the first `mint()` on a fresh deployment reverted**, so it could never be bootstrapped;
- two more defects blocked `redeem()` and every triggered redemption;
- a fourth only appeared under stress: once the reserve ratio fell to the safe threshold, every operation ran out of gas.

Waiting for upstream fixes would have stalled the migration for weeks. Instead I copied the contract into `StablePay/tectonic-local/` and made **the smallest set of changes that let it run**, with a strict rule: no TODO features implemented, no economic parameter or formula changed, nothing touched upstream. Every change was marked `PATCH n` in the source with its reasoning, and the file header listed them all, so a plain diff against upstream shows the patches and little else. I wrote a 10-page record of every modification (25 July) so the maintainers could decide which changes, if any, belonged upstream.

## 2. The patches

| # | Change | Kind | Effect if left as-is |
|---|---|---|---|
| 1 | `ratio()` returns max when liabilities are zero | Defect | First mint on a new deployment always reverts |
| 2 | Removed nested reentrancy guards; split `forceRedemptions` | Defect | `redeem()` and all triggered redemptions revert |
| 3 | Early return in `forceRedemptions` with no holders; re-check inside the loop when the last holder is redeemed mid-iteration | Defect | Division-by-zero panic |
| 4 | Zero guards on `leverage()` and `cushion()` | Guard | Informational reads revert |
| 5 | `ecPrice()` made public | Visibility | Equity-coin price unreadable off-chain |
| 6 | Zero guard on `yieldFromStabilityFeeDaily()` | Guard | Reverts before any equity coin exists |
| 7 | Stability-fee accrual window opens on first charge; fee capped at balance | Defect | A new holder is assessed a fee window back to 1970 (~56 years) |
| 8 | Timestamp written before the burn in `chargeStabilityFee` | Defect | Unbounded recursion; every operation reverts once `r ≤ r_safe` |
| 9 | Constructor validates the paper's parameter constraints | Guard | A mis-set deployment misbehaves silently |
| 10 | Settle a holder's stability fee **before** pricing a redemption and before the sweep reads a balance | Defect | Below critical, any owed fee makes the outer burn fail with `ERC20InsufficientBalance`, reverting every redemption sweep and every mint that triggers one |
| 11 | Gas-capped payouts in forced redemptions; failed payouts credited to `unclaimedBC`, withdrawn with `claimUnpaidRedemption()`; `R()` excludes credited coin | Defect | One holder whose `receive()` reverts or burns gas blocks every sweep |
| 12 | Explicit reverts in `mintEquityCoins` / `redeemEquityCoins` when `E() == 0` | Defect | Panic on mint; redeeming burned coins for nothing |

Patches 1–8 came first (July). Patches 9–12 came in October, when I combined `vm.warp` with a depressed ratio, hostile holders and an exactly-exhausted equity state, and found new failures each time.

Patch 7 was the only one that changed behaviour rather than repairing something clearly unintended. I flagged it explicitly and asked the maintainers to confirm the intended semantics instead of adopting mine.

Cosmetic edits (import paths matching the fork's remapping, the equity-coin symbol `RC` → `EC`, `forge fmt` whitespace, skipping an empty transfer when the treasury fee is zero) were listed separately. I deliberately **didn't** apply the linter's naming suggestions: renaming public variables changes the generated getters and therefore the ABI, and renaming locals would have buried the real changes under hundreds of cosmetic diff lines. The fork's `foundry.toml` suppressed those lints with the reason recorded next to it.

## 3. The test suite as a bug report

Upstream had no tests beyond the Foundry template. I wrote the suite so it would double as a reproducible defect report: tests marked `REGRESSION` fail against upstream and pass against the fork.

| Test | What it proves |
|---|---|
| `test_ChargingAFeeDoesNotRecurseUntilOutOfGas` | Charging a fee is bounded work (asserts a gas ceiling; upstream burns ~1.06 billion gas here) |
| `test_TransferWithAFeeOwedDoesNotRunOutOfGas` | The same defect through an ordinary ERC-20 transfer, i.e. a StablePay direct payment |
| `test_FirstMintSucceedsOnFreshDeployment` | A new deployment can be bootstrapped (upstream: panic `0x12`) |
| `test_FirstEquityMintSucceedsOnFreshDeployment` | Same for equity |
| `test_RedeemSucceeds` | Redemption completes |
| `test_MintBelowCriticalRatioTriggersRedemptionsInsteadOfReverting` | Triggered redemptions actually run |
| `test_ForceRedemptionsWithNoHoldersIsANoop` | Empty holder set handled |
| `test_NewHolderIsNotChargedForTimeBeforeTheyHeldCoins` | No retroactive fee |
| `test_MintPassesTheCheckYetLeavesTheRatioBelowCritical` | Documents that `r ≥ r_crit` after a mint is not an invariant |
| `test_SDK_CostFormulaNeverUnderpaysTheMerchant` | The Solidity side of the SDK's pricing |
| `testFuzz_SDK_CostFormulaNeverUnderpaysTheMerchant` | Same, 512 fuzz runs |
| `testFuzz_LiabilitiesNeverExceedReserve` | Paper Eq. 3 holds (`L ≤ R`, `ratio ≥ 1`), 512 fuzz runs |

The suite grew from 21 tests (July) to 43 (October, with patches 10–12). After my upstream fixes merged, I rewrote it for the synced contract (32 tests, #109). Three tests there are marked `UPSTREAM_BUG` and pin defects still present upstream:

- the sweep reverts when a stability fee is owed;
- a holder that rejects base coin blocks the sweep;
- the sweep panics once holders run out below critical.

Their job is to start failing (in a good way) the day upstream fixes them, so the SDK's defensive guard can be removed.

## 4. Running Tectonic locally and stressing it

### The deploy flow

```bash
anvil
cd StablePay/tectonic-local
forge script script/DeployLocal.s.sol --rpc-url http://127.0.0.1:8545 --broadcast --private-key <anvil key>
```

The script deployed a settable `MockOracle` and Tectonic, seeded 10 ETH of equity so the protocol started over-reserved, printed the addresses and wrote `deployments/local.json` for the SDK and the example app.

A fast sanity check, and the quickest demonstration of the first defect: `cast call $T "ratio()(uint256)"` returned a huge number on the fork and reverted on upstream.

### Pushing the protocol into states a testnet can't give you on demand

The point of a local chain was reaching stressed states whenever I wanted:

```bash
# Raise the oracle price -> liabilities grow -> ratio falls
cast send $O "setPrice(uint256)" 900000000000000 --private-key $K --rpc-url $RPC
cast call $T "ratio()(uint256)"        --rpc-url $RPC
cast call $T "stabilityFee()(uint256)" --rpc-url $RPC   # non-zero once ratio <= 1.5e18

# Let ten days pass so fees accrue
cast rpc evm_increaseTime 864000 --rpc-url $RPC
cast rpc evm_mine --rpc-url $RPC

# Raw vs effective balance of the merchant
cast call $T "balanceOf(address)(uint256)"                  $MERCHANT --rpc-url $RPC
cast call $T "balanceOfAfterStabilityFee(address)(uint256)" $MERCHANT --rpc-url $RPC
```

This is how I checked that the widget showed the stability-fee notice once the ratio dropped to the safe threshold, switched to the forced-redemption warning below critical, and that a merchant's effective balance dropped below the raw balance as fees accrued. It is also how I measured the "mint leaves the ratio below critical" case (r = 1.10 → 1.056 after a 5 ETH mint) and reached the zero-equity state that led to Tectonic #25.

### An 8-step verification checklist

I wrote a runbook where each step says what "passed" looks like, so a failure points at one layer:

1. Contracts compile.
2. Contract tests pass, including the regressions.
3. SDK unit tests pass with no chain.
4. Local deployment succeeds and writes `local.json`.
5. CLI sanity checks: `scPriceMint`, `totalSupply`, `ratio`, `symbol`.
6. End-to-end mint through the SDK (`npm run smoke`).
7. Pay through the widget in MetaMask, once in native coin and once in stablecoin.
8. Exercise the stressed states above.

## 5. End-to-end smoke tests

Unit tests validate the SDK's pricing against a *model* of the contract. The smoke test validates it against **deployed bytecode**. `smoke-local.mjs` read the local deployment, quoted an invoice through the SDK's off-chain math, minted to a merchant address and asserted the merchant received at least the invoice: 14 checks.

Reference output from a clean local deployment:

```
Quote
  100 SC costs 0.0510204 ETH
Mint
  ok    gas was estimated, not hard-coded  gas=254326
  ok    transaction succeeded  gasUsed=164570
  invoiced 100, received 100
  ok    MERCHANT WAS NOT SHORT-CHANGED  surplus=6000 base units
```

- **0.0510204 ETH** is 100 SC × 0.0005 ETH = 0.05 ETH before fees, divided by 0.98 for the 1.5% protocol and 0.5% treasury fees in that local configuration: exactly the inverse of what `mint()` does.
- **6,000 base units of surplus** is 6 × 10⁻¹⁵ SC on a 100 SC invoice, from the round-up plus the 1-wei margin. The merchant is over-credited by dust rather than under-credited at all, which is the direction I wanted.

For the public deployment I replaced it with `smoke-sepolia.mjs`, which checks the live contract read-only by default and can optionally perform a real mint.

## 6. Debugging the Sepolia deployment

The deployment itself (30 October) took more debugging than the code.

| Problem | What happened | How I resolved it |
|---|---|---|
| **First broadcast reverted out of gas** | Sepolia activated the Glamsterdam upgrade on 6 October 2026. Its EIP-8037 charges state gas for each byte of deployed code. Foundry 1.5.1 still simulated with the old costs and set gas limits of 420,750 (oracle) and 4.08M (Tectonic), while Sepolia's own estimates were about 2.03M and 21.4M. The oracle deployment reverted, then the Tectonic deployment reverted because the oracle didn't exist. Cost: about 0.0000004 ETH; the 0.01 ETH seed came back with the revert, but nonces 1 and 2 were used, so final addresses differ from the dry run. | Re-ran with `--gas-estimate-multiplier 800`, raising the limits above the real cost. Both transactions succeeded. Under EIP-8037 the state gas doesn't count toward the 16.7M per-transaction cap, so a 21.3M-gas deployment is valid. The exact command is recorded in `Deployments.md`. |
| **RPC connection resets** | The public Sepolia RPC reset the connection on my home network, so the terminal couldn't reach Sepolia. | Switched to a mobile hotspot, where the same endpoint worked. |
| **No faucet funds** | The Google Cloud and Alchemy Sepolia faucets require a mainnet balance (0.001 and 0.01 ETH), so I couldn't fund a fresh deployer wallet. | Deployed from my existing wallet, which had 0.018 Sepolia ETH, and lowered the seed from 0.05 to 0.01 ETH. |
| **Default RPC alias** | The repository's `sepolia` alias pointed at `rpc.sepolia.org`. | Switched the alias to the publicnode endpoint. |

Before broadcasting I had already rehearsed the full deploy on a **local fork of Sepolia** reading the live Chainlink feed, run `mint`, `redeem` and `mintEquityCoins` against it, and dry-run from the real deployer wallet. After deploying, I read back `oracle()`, `equityCoin()`, `dev()`, `devFeeReceiver()`, `R()`, `ratio()` and `scPriceMint()` from the chain and verified all three contracts on Etherscan.

I wrote the deployment up as a 7-page report: contracts, configuration, read-back state, repository changes, tests, decisions, issues and next steps.

## 7. Smaller experiments

- **Contract-exact rounding.** Folding the contract's two fee deductions into one division gives results that can differ from the contract by 1 wei. Since the merchant guarantee depends on matching exactly, `stablecoinsForPayment` reproduces the contract's two separate floor divisions.
- **Float traps.** `Number("0.1") + Number("0.2")` already loses precision, so `toBaseUnits` parses decimal strings with integer arithmetic, and from #109 rejects JS numbers that aren't safe integers.
- **The 1e24 vs 1e18 trap.** A Djed-scaled fee fed into the Tectonic pricing must throw instead of returning a nonsense quote. A regression test covers this across all three pricing functions.
- **Stale quotes.** Switching network or token quickly could let an older quote overwrite a newer one. A cancellation flag now discards superseded runs.
- **Merge-order analysis.** Before submitting `fix-1-fresh` and `fix-4-6-fresh` I used `git merge-tree` to confirm neither conflicted with the other or with `main`, and documented what would break if only one of them were merged.
- **Responsive rendering.** For the website I rendered the page in Chromium at 12 widths and measured horizontal overflow at each, which found three layout bugs that weren't visible at desktop sizes.
- **Brand rasters.** Rather than exporting favicons by hand, I wrote a script that generates every PNG and ICO size from the SVG logo, and unit tests that check names, sizes, colour modes and ICO frames.
