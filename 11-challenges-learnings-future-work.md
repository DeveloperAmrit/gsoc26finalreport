# 11. Challenges, Learnings and Future Work

## Challenges

### Integrating against a contract that didn't work yet

The biggest challenge was that the migration target wasn't ready. In July there was no Tectonic deployment, and the draft contract couldn't be bootstrapped. Each defect looked like a bug in my own SDK at first: a transaction that reverts with no reason, or a call that never returns because it runs out of gas inside an internal recursion. I handled it in three steps:

1. **Reproduce in isolation.** Copy the contract locally and write a failing Foundry test per defect, so it's clear the problem is in the contract and not in StablePay.
2. **Patch minimally and document.** The smallest fix that unblocks integration, marked and explained, with no design changes slipped in.
3. **Upstream in reviewable pieces.** Three focused PRs (#22, #24, #25) instead of one large one, each with the reasoning, security notes and merge-order analysis a reviewer needs.

### Protocol design questions disguised as bugs

Some issues had no single right answer: what a brand-new holder's stability-fee window should be, whether zero equity should be prevented or merely handled, whether fees apply on transfers. I learned to separate *defects* (clearly unintended, fix them) from *semantics* (flag them, propose an option, let the maintainers decide). The zero-equity question is the clearest example: my first answer handled the state; the merged answer, the super-critical floor, prevents it.

### Reviews and changing direction

Several PRs went through more than one shape: caching (#69 → #71), filters (#72 → #73), Djed removal (#107 → #110), the zero-denominator fix (per-call guards → seeded supplies) and zero equity (guards → super-critical floor). Each time I rebranched from current `main` instead of piling commits onto a stale branch, and wrote down why the new version replaced the old one.

### Deploying on a chain that had just changed its gas rules

The Sepolia deployment failed on the first attempt because the chain's gas rules (EIP-8037, Glamsterdam) were newer than my tooling. Diagnosing it meant comparing Foundry's simulated limits with the network's own estimates and reading the EIP to confirm that a 21.3M-gas deployment was legal under the new rules. Add a home network that reset RPC connections and faucets that required a mainnet balance, and the deployment needed as much persistence as code.

### Infrastructure limits in a backend-less app

StablePay and its dashboard have no backend, so every limit of public RPC providers becomes a product problem: pruned log history, rate limits, default endpoints that reject free requests. Progressive fetching, block caps, lazy per-block timestamps, a permanent cache and an archive-capable gateway with a fallback were all responses to that.

## What I learned

- **Reading a protocol paper against its implementation.** Mapping equations to code line by line is how I found every divergence (fees on transfers, constant vs. parameter, missing constructor checks).
- **Solidity safety in practice.** Checks-effects-interactions matters even when the callback is internal and no external contract is involved. Nested reentrancy guards can lock out your own calls. Every denominator needs a story for why it can't be zero.
- **Fixed-point money in JavaScript.** Bigints everywhere, decimal strings at the boundary, rounding in the merchant's favour, matching the contract's division order exactly, and verifying the minted amount from the receipt instead of trusting the quote.
- **Foundry beyond `forge test`.** Fuzzing invariants, `vm.warp` with mocked oracles to reach stressed states, fork rehearsals against live feeds, deploy-script tests, gas reports and Slither in CI.
- **Writing for reviewers.** Small PRs, a clear statement of scope and of what is *not* included, security and storage-layout notes, test evidence, and merge-order analysis. Mentors could merge contract changes because they could see exactly what changed and why.
- **Documentation as a product.** Writing a 10-step tutorial and verifying every sample against the live deployment surfaced the underpayment edge case that led to mint verification in the SDK.

## Future work

### StablePay

- Merge the open PRs: #108 (mid-term files and hardening) and #111 (Sepolia).
- Publish a new `stablepay-sdk` release to npm with the Tectonic-only SDK, and publish `tectonic-sdk` as its own package.
- **React Native variant of the widgets**, so mobile merchant apps can embed StablePay.
- Remove the doomed-mint guard in `tectonic-sdk` once the forced-redemption defects are fixed upstream.
- Decide StablePay's revenue model, since Tectonic has no equivalent of Djed's UI fee.
- Add networks as Tectonic deploys beyond Sepolia (Base Sepolia, then mainnets).

### Tectonic

- Merge #27 (Sepolia deployment) and decide who holds the `dev` role long term (`proposeDevTransfer` / `acceptDevTransfer`).
- Add equity to the Sepolia deployment: with a 0.01 ETH seed, small mints already reach the stability-fee and critical thresholds.
- Fix the three open forced-redemption defects (fee owed during the sweep, holders that reject base coin, running out of holders below critical). The fixes and tests already exist in my local history.
- Port a full test suite upstream; the deploy and oracle tests are a start.
- Resolve the open design questions: fees on transfers, the post-mint ratio check, `f_max` as a parameter, `forceRedemptions` economics.
- Plan mainnet oracles. Ethereum Classic has no Chainlink feed, so it needs a different source (Djed's adapter family or OrbOracle).
- Upgrade Foundry once it models Glamsterdam gas, so deployments no longer need a manual multiplier.

### Merchant dashboard and website

- Merge #75 (UI refresh), #76 (Tectonic transactions) and #83 (documentation).
- Show effective balances (`balanceOfAfterStabilityFee`) and forced-redemption events to merchants.
- Keep the docs in sync with the SDK, and re-run the tutorial end to end whenever the contracts or SDK change.

## Acknowledgements

Thanks to my primary mentor **Tanya Ruby**; to **Bruno** for reviewing and merging the Tectonic contract work, for the design discussions that shaped the seeded supplies and the super-critical floor, and for two rounds of website feedback; to **Sarthak** for reviewing and merging the StablePay and website PRs; to the rest of the mentor team (Blizet, Yogesh, Nidhi, Raj and Aditya Gupta); and to **Atharva Naik**, my pair-programming peer, for reviewing the Tectonic adapter.
