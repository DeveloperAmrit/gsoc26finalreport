# 10. Community Bonding and Preparation

## 1. Community bonding (May 2026)

### Revising the plan

- Revised the proposed timeline with the mentors. The original plan included building support for several Djed variants; I dropped that and planned time to study Tectonic, since StablePay was going to move to it.
- Got the PRs from the application phase reviewed.
- Reviewed open issues and pull requests in the StablePay repositories to plan the first six weeks.

### OrbOracle

OrbOracle is Stability Nexus's decentralized oracle. Working on it during community bonding taught me how price data reaches a stablecoin protocol on-chain, which turned out to matter later when choosing an oracle for the Tectonic deployment.

**OrbOracle-Poster #2: automated price-poster service** (merged May 12, fixes #1, +443 lines)

A background service that continuously posts prices to OrbOracle:

- **Deployment:** a `Dockerfile` and `docker-compose.yml` so the worker can run 24/7.
- **Multi-source aggregation:** fetches the price concurrently from three independent providers: CoinGecko (REST), Chainlink (on-chain contract read) and Pyth Network (Hermes API).
- **Data cleaning:** a median filter to suppress outliers and failed sources, then conversion to the BigInt ×1e8 format the Orb contracts expect, avoiding floating-point errors.
- **Pre-flight stake check:** reads the operator's staked tokens and halts before sending a transaction if the stake is below the required minimum, so the operator doesn't waste ETH on a submission that would be rejected.
- **Gas-aware triggers:** skips posting when base fees spike.

**OrbOracle-Poster #5: README** (merged May 16): rewrote the README with everything an operator needs to run the poster.

**OrbOracle-Poster #6: multi-coin support** (merged May 27): the configuration defaults to Ethereum Classic; the CoinGecko asset is configurable through an environment variable; example RPC endpoints and Pyth/Chainlink identifiers updated; the stake check now adds locked and unlocked token balances instead of relying on a single total.

**OrbOracle-EVM-Frontend #39: decimal-scaling fix** (merged May 3): while testing the poster, the frontend showed a value of 0 for a legitimate submission. I traced it to a `PRICE_DECIMAL` mismatch and fixed it.

**OrbOracle-EVM-Frontend #40: ETC factory address** (opened May 26, merged Oct 8): the previous `OracleFactory` deployment on Ethereum Classic assumed the chain supports the `MCOPY` opcode, as Ethereum and Scroll Sepolia do. ETC doesn't, so I pointed the frontend at a working deployment.

### PictoPy

**AOSSIE-Org/PictoPy #1269: watcher-service deadlock** (May 20, fixes #1031). When a watched folder was deleted, PictoPy's sync microservice tried to restart its file watcher *from the watcher thread itself*. The stop function called `watcher_thread.join()`, so the thread was joining itself: a deadlock that hung for the 5-second timeout and then raised an error. My fix only joins when the current thread isn't the watcher thread, and when a restart is requested from the watcher thread it spawns a short-lived daemon thread to perform the stop/start, letting the original thread exit cleanly. The PR was closed without merging.

## 2. Application phase (December 2025 to April 2026)

Before GSoC started, I contributed across AOSSIE and Stability Nexus projects to learn the codebases and the review process. The StablePay-related ones fed directly into the project.

### StablePay and Djed

| PR | What | Status |
|---|---|---|
| StablePay #33 | Integrated Gluon into StablePay so it could accept payments through a Gluon stablecoin | Closed |
| Gluon-EVM #9 | Router for the fission step of the StablePay–Gluon integration | Closed |
| StablePay #39 | Rebuilt the SDK `dist` for the latest changes and bumped `stablepay-sdk` to 1.0.4 | Merged |
| StablePay #53 | npm publish workflow for both SDKs | Open |
| StablePay #81 | Upgrade the Djed SDK for advanced protocol features and Djed Shu | Closed |
| Merchant Dashboard #35 | Show only transactions involving the merchant | Closed |
| Merchant Dashboard #36 | Search on the transactions page | Merged |
| Merchant Dashboard #37 | Filters (first attempt) | Closed |
| Merchant Dashboard #38 | Made the dashboard responsive | Merged |
| Merchant Dashboard #44 | Pagination and infinite scroll (first attempt) | Closed |
| Merchant Dashboard #53 | Handle refs in the Button component with `React.forwardRef` | Merged |
| Merchant Dashboard #54 | Export as CSV or JSON (first attempt) | Closed |
| Landing Page #51 | Make the navbar functional | Open |
| Djed-Solidity #47 | Configurable stablecoin and reserve-coin names and symbols | Open |
| Djed-Solidity #48 | Clean up and modernize the unit-testing setup | Open |

Several of these (search, filters, pagination, export) were early versions of issues I solved properly during the coding period, once I understood the dashboard's data flow and RPC constraints.

### Other Stability Nexus projects

| Project | PRs | Highlights |
|---|---|---|
| Maelstrom-Solidity | #41 (merged), #48 | Fixed the exponential decay calculation in `_getDecayValue`; proposed reentrancy protection on critical external functions |
| Maelstrom-WebUI | #23, #25 (merged), #24, #26 | Horizontal scrollbar, navbar/footer theme, hover and readability fixes |
| HammerAuctionHouse-WebUI | #49 (merged) | Navbar active-page indicator across all pages |
| Zplit | #22 (merged) | Website overhaul: folder structure, light mode and toggle, responsiveness, animations, working links |
| Chainvoice | #64, #65 | GitHub Pages 404-on-refresh fix and query-parameter handling |
| IdentityTokens-EVM-Contracts | #15 | Identity Tokens blueprint with frontend and tests |
| Website | #31 | Footer component |

### Other AOSSIE projects

| Project | PRs | Highlights |
|---|---|---|
| AOSSIE Website | #610, #612 (merged), #553 | Revamp of the AOSSIE website |
| Resonate-Website | #75, #266, #268 (merged) | Replaced static image downloads with components, SVG tech-stack icons, footer copyright |
| DebateAI | #162, #168 (merged), #165, #166, #167, #197, #198 | Concede option against bot and player, a notification system, profile popup, custom popups, gamification |
| BabyNest | #147, #155 (merged), #136, #144–#146, #148, #154 | Navigation and navbar fixes, docs fixes, country autofill, edit profile |
| Ell-ena | #134 (merged), #133, #135 | Signup page UI, `pg_cron` extension, calendar focus-day bug |
| Rein | #34 (merged), #32, #33, #35 | WebSocket initialisation and CSS in production builds, authentication, LAN IP detection, mobile scrolling |
| PictoPy | #967, #990 | Duplicate and best-shot detection, celebrity tagging |
| OrgExplorer | #14, #16 | Next.js + Tailwind v4 migration, initial UI |
| Gitcord | #2 | Basic GitHub–Discord bot |
