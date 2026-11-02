# 07. Website and Documentation

[DjedAlliance/StablePay-LandingPage](https://github.com/DjedAlliance/StablePay-LandingPage) is the StablePay website (Next.js, TypeScript, Tailwind). During the project I redesigned it, took it through two rounds of mentor feedback, and added a full documentation site to it.

## 1. The redesign (PR #82, merged Oct 31)

### Why

The old site used a dark "space" theme with a starfield, a generic placeholder logo and copy that described StablePay as a Djed product. StablePay was moving to Tectonic, Tectonic's own web interface is light, and the StablePay logo's colours were not used anywhere on the page.

### Design system

I built the new look from the two brands so StablePay and Tectonic read as one family.

| Colour | Hex | Role |
|---|---|---|
| Blue | `#235EFE` | Primary, from the StablePay logo |
| Gold | `#FFC822` | Accent, matched to Tectonic Gold `#FFC517` |
| Orange | `#FD6724` | Accent, from the StablePay logo |
| Forest | `#228B22` | Health and success, from the Tectonic brand |
| Ink | `#0A1633` | Headings, a deep navy instead of black |

- `tailwind.config.ts`: full colour scales for each brand colour, fluid display type sizes, shadow and motion tokens.
- `globals.css`: a light base, the logo's three-circle motif as a subtle background wash, card and grid primitives, and reduced-motion handling.
- Inter for text and JetBrains Mono for code; the serif display faces were dropped.
- Shared primitives extracted: `Button`, `Logo`, `Reveal`, `SectionHeading`, and a `lib/site.ts` for links and copy shared across sections.
- The real three-circle logo mark replaced the placeholder; a matching favicon was added.
- Page metadata, Open Graph tags and a skip-to-content link.

### Content

Copy is Tectonic-only. New and rewritten sections:

- **Hero** with a static mock of the checkout widget instead of a starfield.
- **How it works**: a diagram of both payment flows (pay in stablecoin, pay in native coin and the merchant receives stablecoin).
- **Developers**: a tabbed code sample covering install (npm, pnpm, yarn) and integration.
- **Tectonic**: an explainer of the protocol behind StablePay.
- **Networks**: a status table of supported networks.
- **FAQ**.
- Rewritten Header, Features, Call to action and Footer; the logo ticker replaced by a trust bar.

### Responsive testing

I rendered the page in Chromium at 12 widths (320, 375, 414, 480, 640, 768, 900, 1024, 1120, 1280, 1440 and 1920 px) and fixed what broke:

- The networks table and a `<pre>` block were setting their grid column's minimum width, so the whole page scrolled sideways below 414 px. Grid items default to `min-width: auto`; adding `min-w-0` let their own overflow containers scroll instead.
- The four-node payment-flow diagram never fit a half-width column, so the two flows stack and only go horizontal from the `lg` breakpoint. This fixed an overflow at 1024 px.
- Two floating badges on the widget mock covered its own amount and fee rows at every width; I removed them.
- The code card got a minimum height so switching tabs doesn't make the layout jump.

The result: zero horizontal overflow at all 12 widths, no console errors, and clean `tsc --noEmit` (strict), `next lint` and Tailwind builds.

### Mentor feedback

Bruno reviewed the redesign twice.

**First round (Oct 17):** content changes across most sections, a terms-of-use link, updated social icons (GitHub, X, YouTube), a Stability Nexus mark in the footer, and a cleaner trust bar.

**Second round (Oct 31):**

- the widget mock says "Ethereum" instead of "Ethereum Sepolia";
- reworded the "How it works" description and added line breaks in headings;
- removed Milkomeda C1 from the networks list;
- dropped the "Djed successor" wording from the Tectonic introduction;
- rewrote the peg card without the redemption formula.

The PR was merged the same day.

## 2. The documentation site (PR #83)

StablePay had no developer documentation beyond READMEs. I wrote 16 pages and built a `/docs` section into the website to host them. The docs talk only about StablePay and Tectonic.

### Pages

| # | Page | Section | What it covers |
|---|---|---|---|
| 01 | Welcome to StablePay | Get started | What StablePay is and isn't, the two ways to pay, where Tectonic fits, where to go next |
| 02 | Quick start | Get started | The shortest path from `npm install` to a working checkout |
| 03 | Thinking in StablePay | Get started | The mental model: a checkout is just a transaction |
| 04 | How a payment works | Learn | Both payment paths, step by step, with a sequence diagram |
| 05 | Meet Tectonic | Learn | Reserve, liabilities, reserve ratio, stability fees, triggered redemptions, in merchant terms |
| 06 | Getting paid | Learn | What a merchant receives, effective balances, what happens when reserves are low |
| 07 | Tutorial: Add StablePay to your store | Guides | A 10-step tutorial: create a store, install, add the checkout, take a first payment, create orders on a server, verify on-chain, connect widget to server, test end to end, go-live checklist |
| 08 | Verifying payments | Guides | Why the browser's word isn't enough; Node.js and Python verifiers; confirmations; one payment per order; handling underpayments; watching an address |
| 09 | Next.js | Guides | Integration in a Next.js app |
| 10 | Any website | Guides | Integration in a plain website |
| 11 | Customizing the checkout | Guides | Styling and configuration |
| 12 | Widget API reference | Reference | Every component, prop and callback, including `onTransactionUnderpaid` |
| 13 | Tectonic SDK reference | Reference | `TectonicClient`, pricing functions, units and rounding |
| 14 | Networks & contracts | Reference | The Sepolia deployment, addresses and parameters |
| 15 | Troubleshooting & FAQ | Reference | Common errors and what they mean |
| 16 | Glossary | Reference | Terms used across the docs |

Every code sample in the guides was run against the live Sepolia deployment when it was written.

### How the docs site works

- The pages are plain GitHub-flavoured Markdown with front matter (title, description, sidebar section). They read correctly on GitHub as well as on the site. The numeric prefix sets the order and is dropped from the URL (`07-integration-guide.md` → `/docs/integration-guide`).
- Links between pages are relative file names, which work on GitHub and are rewritten by the site.
- Callouts use GitHub's alert syntax (`> [!NOTE]`, `> [!WARNING]` and so on). "Deep dive" and "challenge" sections are `<details>` blocks. "You will learn" and "Recap" headings are styled as boxes.
- **Interactive widgets** are written as HTML comments such as `<!-- widget:live-stats -->`. GitHub hides them; the site renders the widget. A Mermaid block placed right after a widget comment is the GitHub fallback and is replaced by the widget on the site.

Components I built for it:

- A custom Markdown renderer, a code block with titles and a copy button, callouts, deep-dive blocks and sequence diagrams.
- A docs sidebar, a per-page table of contents and full-text **docs search**.
- Six widgets, plus a ratio gauge they share:
  - `checkout-preview`: a preview of the checkout widget;
  - `payment-flow`: a step-by-step walk (quote, ask, pay) between the StablePay widget, the customer's wallet, the Tectonic contract and the merchant's address;
  - `quote-calculator`: how much ETH a given invoice in SC costs, using the same round-up formula as the SDK and the live fee and dev fee;
  - `live-stats`: reads the reserve, stablecoin supply, price of 1 SC, fees, daily stability fee and reserve band **from the Sepolia deployment** in the browser;
  - `reserve-playground`: move the reserve ratio (with presets like "comfortable", "getting tight" and "critical") and see how a held balance shrinks under stability fees and when redemptions kick in;
  - `tutorial-map`: a checklist of the tutorial's 10 steps with progress, linking to each step;
  - the ratio gauge marks the super-critical floor, the triggered-redemption zone, the stability-fee zone and the healthy zone.

The site also updates the landing page's deployment information to point at Sepolia.

## 3. Video tutorials and learning resources

- **Video tutorials (week 18).** I recorded tutorials showing how to integrate StablePay into a merchant website, and added them to the StablePay website.
- **Learning resources (week 19).** Material a new contributor needs was scattered across papers, repositories and websites. I gathered resources on Djed, Gluon, OrbOracle and related projects into one section of the StablePay website.
