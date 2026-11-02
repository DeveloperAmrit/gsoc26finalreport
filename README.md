# GSoC 2026 Final Report: StablePay

**Contributor:** Amrit Kumar Yadav ([@DeveloperAmrit](https://github.com/DeveloperAmrit))
**Organization:** AOSSIE (sub-organization: Djed Alliance / Stability Nexus)
**Project:** StablePay (Large, 22 weeks)
**Primary mentor:** Tanya Ruby

This folder is the written record of my Google Summer of Code 2026 work. It covers every pull request I opened, the experiments and local tooling I built along the way, and the research behind the Djed-to-Tectonic migration. The PDF in this folder (`GSoC2026_Final_Report_Amrit_Kumar_Yadav.pdf`) is the short, submitted version. These Markdown files are the long version.

---

## Contributor details

| Field | Value |
|---|---|
| Name | Amrit Kumar Yadav |
| GitHub | [github.com/DeveloperAmrit](https://github.com/DeveloperAmrit) |
| Discord | Amrit Kumar Yadav \| DeveloperAmrit |
| Email | kittyamrit005@gmail.com |
| Portfolio | [developeramrit.vercel.app](https://developeramrit.vercel.app) |
| Pair-programming peer | Atharva Naik |

## Project details

| Field | Value |
|---|---|
| Organization | AOSSIE, sub-organization Djed Alliance / Stability Nexus |
| Project title | StablePay |
| Size | Large, 22 weeks |
| Primary mentor | Tanya Ruby |
| Mentors | Bruno, Blizet, Sarthak, Tanya, Yogesh, Nidhi, Raj, Aditya Gupta |

### Repositories I worked in

| Repository | What it is |
|---|---|
| [DjedAlliance/StablePay](https://github.com/DjedAlliance/StablePay) | The embeddable payment widget (`stablepay-sdk`), the protocol SDK (`tectonic-sdk`), example app |
| [DjedAlliance/StablePay-MerchantDashboard](https://github.com/DjedAlliance/StablePay-MerchantDashboard) | Next.js + TypeScript dashboard where merchants read the payments they received |
| [DjedAlliance/StablePay-LandingPage](https://github.com/DjedAlliance/StablePay-LandingPage) | The StablePay website and, from this project, its documentation site |
| [StabilityNexus/Tectonic-EVM-Contracts](https://github.com/StabilityNexus/Tectonic-EVM-Contracts) | The Tectonic stablecoin protocol in Solidity (Foundry) |

---

## The project in one paragraph

StablePay is a fully decentralized checkout. A merchant drops the StablePay widget into a website; the customer signs a single transaction in their wallet; the widget talks straight to the stablecoin protocol's smart contracts with no server, payment processor or custodian in between. A customer can pay in stablecoins, or in the chain's native coin, in which case the protocol mints stablecoins directly to the merchant in the same transaction. StablePay was built on the Djed protocol. My project made StablePay production-ready for merchants (widgets, dashboard), then **migrated it from Djed to Tectonic**, Stability Nexus's successor protocol. Tectonic was still a draft contract when I started, so the migration turned into finding and fixing protocol defects upstream, deploying Tectonic to a public testnet myself, and then pointing StablePay at that deployment.

## Outcomes at a glance

- **24 pull requests** opened during the coding period across four repositories, **15 merged**, plus **6** during community bonding.
- **StablePay SDK:** redesigned widgets, stronger validation and error handling, an in-repo example site, a protocol-adapter layer, a new `tectonic-sdk` package, and a full switch to Tectonic-only.
- **Tectonic contracts:** found and fixed five defects that made the contract unusable (including an unbounded recursion that burned ~1.06 billion gas), seeded supplies so ratio denominators can never be zero, added a super-critical reserve-ratio floor, reworked the fee recipient model, wrote a Chainlink inverting oracle, and **deployed and verified Tectonic on Ethereum Sepolia**.
- **Merchant dashboard:** pagination, sorting, progressive fetching, permanent cache, filters, CSV export, multiple merchant addresses, a UI refresh, and Tectonic transaction support.
- **Website:** a full redesign on the StablePay and Tectonic brand palettes, two rounds of mentor feedback, and a 16-page documentation site with interactive widgets.
- **Research:** a written assessment of the Tectonic draft, a Djed-to-Tectonic function mapping, a tiered list of 23 questions for the protocol maintainers, and several protocol-level observations that fed into design changes.

## Files in this report

| File | Contents |
|---|---|
| [01-project-overview.md](01-project-overview.md) | StablePay, Djed and Tectonic; what the project set out to do and how it changed |
| [02-timeline.md](02-timeline.md) | Week-by-week plan and a dated chronology of what actually happened |
| [03-pull-requests.md](03-pull-requests.md) | Every pull request, with status, size and a description of each |
| [04-stablepay-sdk.md](04-stablepay-sdk.md) | Widget, SDK and protocol-adapter work in the StablePay repository |
| [05-tectonic-contracts.md](05-tectonic-contracts.md) | Defects fixed, design changes, oracle and the Sepolia deployment |
| [06-merchant-dashboard.md](06-merchant-dashboard.md) | Merchant dashboard features and performance work |
| [07-website-and-documentation.md](07-website-and-documentation.md) | Website redesign and the documentation site |
| [08-research.md](08-research.md) | Protocol study, contract assessment, integration plan, questions for maintainers |
| [09-tinkering-and-experiments.md](09-tinkering-and-experiments.md) | The local Tectonic harness, test suites, stress experiments, deployment debugging |
| [10-community-bonding-and-preparation.md](10-community-bonding-and-preparation.md) | Work before the coding period: OrbOracle, PictoPy and the application phase |
| [11-challenges-learnings-future-work.md](11-challenges-learnings-future-work.md) | What was hard, what I learned, and what is left |
