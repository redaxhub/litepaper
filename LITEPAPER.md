# REDAX Hub — Litepaper

**Solana-native Merger-as-a-Service infrastructure for campaign-based token consolidation and asset migration.**

---

**Version:** 1.0  
**Date:** 2026-04-26  
**Authors:** Tuncer Timur, Ahmet Akın Aktan  
**Status:** Public draft for community review.

---


## What is REDAX Hub?

REDAX Hub is a Solana program that lets any project run a token migration campaign through a single audited factory, instead of writing a custom contract and paying for a dedicated audit each time.

A project opens a campaign, defines what old tokens it accepts and at what rate, and users convert their old tokens into the new one through one program. The conversion is atomic and the math is precise. The platform charges a small fee on the new token plus a flat campaign creation fee in SOL.

We call the category **Merger-as-a-Service**. To our knowledge, REDAX is the first protocol to offer it on Solana as a permissionless factory.


## The Problem

Solana has gone through a launchpad era. Since 2024, Pump.fun alone has reportedly launched more than 11 million tokens on the network, and aggregate launch counts across all permissionless launchpads on Solana are even higher. Only a small fraction of these tokens graduate into open-market trading or sustained market activity. Multiple Dune-based market trackers have shown Pump.fun graduation rates around 1-2%, and in some 2025 periods below 1%.

The structural consequences of this distribution are visible across the ecosystem:

**Holders sit on positions they cannot exit.** A user who bought into a project that did not work out is left with an illiquid token they cannot easily trade or use.

**Liquidity pools become inaccessible.** Many of these tokens were paired with SOL on Raydium or Meteora at launch. When trading volume disappears, the SOL inside the pool stays there, and the price impact of any non-trivial trade against a thin pool makes that SOL effectively unreachable.

**Communities scatter.** Some of the projects in this distribution were launched in good faith, hit unfavorable conditions, and would relaunch with cleaner tokenomics if they had a path to bring their original holders along. Today they don't have that path.

**Capital cannot reform around viable ideas.** A team that wants to do a clean v2 has to either ignore its v1 holders or build migration infrastructure from scratch — a cost that prevents most healthy relaunches from happening.

Today, every project that wants to do a token migration faces the same set of tasks: write a custom Rust program, pay for a dedicated audit, build a custom user interface, and coordinate community communication. This typically costs between thirty and one hundred thousand dollars and takes two to four months. Most projects cannot afford it. So the migration never happens, and the community stays trapped.


## Our Solution

REDAX builds the migration factory once, audits it once, and lets any project use it.

The flow is simple.

A project opens a **campaign** by paying 0.5 SOL. They define which old tokens are accepted (one or many), the conversion rate per token, and the maximum input amount per token. They choose how the platform's accumulated fees should be handled (described below). They publish the campaign.

A holder of an old token visits the campaign's page, connects their wallet, and converts. In a single transaction, the old tokens are burned (or, in a future phase, locked in a vault), and the new tokens are minted to the holder's wallet, minus a 1% platform fee. The conversion is atomic — there is no intermediate state where the user has lost their old tokens but not yet received their new ones.

The campaign is fully on-chain. There is no off-chain database. There is no centralized backend. The campaign's state, parameters, and history are all readable directly from Solana RPC.

When the campaign ends, any unclaimed old tokens stay unclaimed. The campaign creator finalizes the campaign and accounting closes out.

That is the whole product.


## Why This Matters Now

Several trends converge in 2026 that make REDAX timely:

**The aftermath of the launchpad era is now visible at scale.** With more than 11 million tokens having been launched on Solana since 2024 and only a small share graduating into sustained activity, the size of the migration-and-consolidation problem is now hard to ignore.

**Token mergers became respectable.** The ASI Alliance merger of Fetch.ai, SingularityNET, and Ocean Protocol, announced in 2024 with fixed conversion rates and a roughly six-billion-dollar fully diluted valuation at the time of announcement, demonstrated that fungible-token mergers at large scale are technically and operationally feasible. Smaller examples followed: Ribbon Finance to Aevo, Merit Circle to Beam, Router v1 to v2. The pattern is established.

**Solana's execution stack continues to mature** through client diversity, priority fee markets, Token-2022 extensions, and deeper liquidity routing. A factory protocol that needs predictable execution and flexible token semantics has a stable base to build on.

**The market wants infrastructure, not memes.** After the launchpad era, capital and grants are rotating toward protocols that solve real coordination problems. Infrastructure-grade Solana projects have demonstrated that there is appetite — and ecosystem support — for tools that compound rather than for hype-driven launches.

A factory for token mergers fits exactly into this current.


## How REDAX Works — Architecture

REDAX is structured as a hierarchy of program-derived addresses (PDAs). Every account is computed deterministically from its seeds. There is no off-chain state.

```
ProtocolConfig (singleton)
  └── Campaign (one per migration)
        ├── TokenConfig (one per accepted legacy token)
        ├── CampaignTreasury (output-token escrow)
        └── DustAccumulator (rounding residue)
```

The protocol-level config holds global parameters. Each campaign is its own isolated state tree. Multiple campaigns can run in parallel without interference.

A single conversion executes this sequence in one atomic transaction:

1. The holder calls `convert_exact_in` with their wallet, the campaign address, the legacy token, and the amount.
2. The program verifies the campaign is active, the token is enabled in this campaign, and the per-token cap will not be exceeded.
3. The program receives the legacy token from the holder. In Phase 1 (burn mode), the tokens are destroyed.
4. The program computes the conversion using `u128` precision math, determines the gross output amount, and calculates the fee.
5. The program mints the net output (gross minus fee) to the holder and the fee to the campaign treasury.
6. The program emits an event capturing the full transaction breakdown for indexers and analytics.

The conversion is irreversible. The user cannot undo it. The campaign cannot reverse it. This is intentional — auditable irreversibility is a security property, not a limitation.


## Precision Sandwich — The Math

Solana programs operate on integers. There is no floating-point arithmetic at the program level. Every token amount is a raw integer scaled by the token's `decimals` field.

When two tokens with different decimals merge, naive division produces truncation loss. We have observed in early DeFi migrations that decimal mismatches, handled incorrectly, can lose meaningful amounts of user value across a campaign.

We address this with a four-stage pipeline we call the **Precision Sandwich**: normalize, scale, fee, denormalize.

The mechanics in summary:

- All intermediate calculations use `u128` arithmetic, never `u64`. This prevents overflow on any realistic input.
- We normalize the input and output to a common decimal precision before applying the rate, then denormalize back to the output's native decimal at the end.
- Rounding direction is explicit. When the rounding favors the user, we floor. When it favors the protocol, we ceil. This is the same convention as ERC-4626 and the OpenZeppelin tokenized vault reference implementation.
- Any sub-unit residue from rounding is captured in a per-campaign dust accumulator, making the conservation of value publicly auditable.

The result is that for any conversion, the following invariant holds:

```
sum(legacy_tokens × rate) = sum(user_output + protocol_fee) + dust_residue
```

No value is created or destroyed by the math. Every fraction of a unit is accounted for somewhere.

The patterns we use are not novel. They follow Uniswap V3's `FullMath`, OpenZeppelin's `Math.mulDiv`, and the ERC-4626 rounding specification. We cite these because audit firms recognize them as well-formed, and because we want to be transparent about where the math comes from.

A future technical document will describe the algorithm in full, with worked examples, invariants, and threat-model discussion.


## Treasury Policy — Where the Fees Go

When the protocol takes a 1% fee on conversions, that fee accumulates in the campaign treasury as the new token. This raises a question: what does REDAX do with these tokens?

Selling them immediately on the open market would create sell pressure on the new token, harming the campaign creator and the holders who just converted. Holding them forever would mean only paper revenue, never realized.

We resolve this by giving the **campaign creator** the choice of treasury policy at the moment they open the campaign. The chosen policy is locked into the campaign and cannot be changed afterward. There are four options:

**Hold Forever.** REDAX never sells. Zero sell pressure on the new token. The trade-off is that REDAX has only paper revenue from this campaign.

**Vest and Controlled Execution (default).** Accumulated tokens enter a vesting cliff (default 90 days after campaign finalization) and are then released gradually over time. Released tokens can be routed through Solana liquidity infrastructure such as Jupiter, with MEV-aware transaction broadcasting where supported. Each individual sale is small enough that price impact is minimized.

**Liquidity Injection.** Accumulated tokens are paired with SOL and added as permanent liquidity to the new token's primary AMM pool. Sell pressure is zero, the new token gets permanent liquidity depth, and REDAX earns LP fees over time.

**Mixed.** A linear combination of the above (for example, 50% Hold, 30% Controlled Execution, 20% LP).

To our knowledge, this policy-by-creator pattern is novel for a Solana factory protocol. Most factories impose a single accumulated-fee strategy. REDAX delegates the decision to the party with the most direct economic interest in the new token — the project that just launched it.


## Fee Model

REDAX collects fees through three channels:

**Campaign creation fee — 0.5 SOL (one-time, per campaign).** Paid by the campaign creator at the time the campaign is opened. Goes directly to the protocol treasury in SOL. This covers the rent for PDA creation and prevents low-effort spam.

**Output fee — 1% (per conversion).** Paid in the new token, on every conversion. Accumulates in the campaign treasury and is managed under the chosen treasury policy.

**Optional convert fee — 0.001 SOL (Phase 2, anti-spam).** A small SOL fee on each conversion, activated only if we observe DDoS-style abuse. Sub-cent in normal usage.

REDAX has not minted its own token. There is no governance token. There is no TGE. There is no fundraising round in progress. The protocol is funded entirely by the SOL creation fees and the realized portion of the output-token fees. We may launch a token in the future, but we have no commitment to do so and no planned date.

This is the path Jupiter took: years of operation as a fee-generating product before any token launch. We believe this discipline produces a better protocol.


## Phased Rollout

REDAX ships in three phases. Each phase has explicit gating criteria.

**Phase 1 — Burn-only MVP.** Everything described in this document, with one omission: legacy tokens are destroyed at conversion time, not held in a vault. This is the simplest, safest, most auditable version of the protocol. Target deployment: late Q3 2026 (subject to audit completion).

**Phase 2 — Hardening.** Verified tier (a UI distinction for campaigns that pass a baseline trust check), Token-2022 extension allowlist registry, rate updates with timelock and holder opt-out, custom indexer for the discovery UI, and bug bounty program. Target: Q4 2026.

**Phase 3 — Lock and Liquidity Recovery.** Lock mode introduces a vault where legacy tokens are held instead of burned. Crank operations could then route accumulated legacy tokens through Solana DEX infrastructure to recover SOL from low-activity liquidity pools. The technical mechanism is designed; the legal and regulatory framework around it is not yet settled.

**Important note on Phase 3.** This phase raises questions — about asset ownership in the lock state, about how crank operations are classified across jurisdictions — that we have not resolved. We are working with external counsel to clarify these. **Phase 3 is fully gated on the resolution of these legal questions. If the legal risk is determined to be unmanageable, Phase 3 will be removed from the protocol or substantially redesigned.**

We describe Phase 3 here in transparency about our long-term intent. We do not commit to its activation.


## Security Approach

The arithmetic of REDAX is the most security-critical component of the system. A bug here would corrupt every conversion. We have over-engineered this layer specifically to be auditable and provably correct.

We have studied the public audit reports of established Solana auditors operating on similar program archetypes. Several common findings — strict mint validation, rent-exempt calculations, token-program interface unification, slippage guards on DEX CPI, conservation-of-value invariants — were addressed at the architectural level before any code was written.

Our audit strategy is tiered:

**Internal hardening (continuous).** Rust `checked_*` arithmetic everywhere; Anchor's account constraint validation; property-based testing with proptest; fuzz testing on the conversion module; verifiable build in CI.

**Tier 2 — Competitive audit (pre-mainnet).** A time-boxed contest on Sherlock or Code4rena, where dozens of auditors review the code in parallel. Cost in the range of ten to twenty-five thousand dollars.

**Tier 3 — Boutique audit (mainnet readiness).** A focused review by a Solana-specialized firm such as CoinFabrik or OtterSec. Cost in the range of fifteen to fifty thousand dollars.

**Tier 4 — Premier audit (post-traction).** A full-scope audit by a top-tier firm, funded ideally by a Solana Foundation grant or seed investment.

**Continuous bug bounty (post-mainnet).** Hosted on Sherlock Bug Bounty or Immunefi, with the reward pool funded from the protocol treasury.

We do not assume external audit funding will be available before Phase 1 mainnet. The earlier tiers are scoped to be funded from personal contributions, micro-grants, and the protocol's own SOL creation revenue during the guarded beta phase.


## What We Are Not

REDAX is not a launchpad. We do not help projects do their initial token launch.

REDAX is not a wallet cleaner. We do not help users batch-burn dust tokens for rent SOL. (Sol-Incinerator does that well; we are not competing with them.)

REDAX is not a DEX. We do not provide spot trading. We use Solana liquidity infrastructure such as Jupiter for any routing we need.

REDAX is not an aggregator. We do not aggregate liquidity across other protocols.

REDAX is not, in Phase 1, an attempt to recover SOL from dead pools. That capability would belong to Phase 3, which is gated on legal clarity and is not committed.

REDAX is one thing: a campaign-based factory for converting old tokens into new ones, with precise math, atomic execution, and auditable accounting.


## Comparison to Prior Work

We did not invent the idea of a token merger. We are generalizing patterns that have been demonstrated at the project-by-project level.

**ASI Alliance (2024).** Three projects merged into one through audited migration contracts. Conversion rates were fixed in advance. Demonstrated that multi-billion-dollar fungible-token mergers are feasible. Each migration required its own bespoke contract and audit. REDAX is what you get when you build that contract once.

**Pak's Merge (2021).** Murat Pak's NFT-based merger experiment was sold via open-edition auction and demonstrated that merger mechanics can be made legible and desirable to non-technical participants. Conceptual prior art for the user experience, not for the technical implementation.

**Ribbon to Aevo (2023).** Ribbon DAO approved the merger of Ribbon Finance into Aevo. The transition used a 1:1 swap structure and is one of several precedents for orderly v1-to-v2 token transitions on EVM chains.

**Existing Solana factories.** Streamflow demonstrates that Solana teams already adopt audited, reusable infrastructure for token operations such as vesting, locks, airdrops, and payroll. Squads provides multisig-as-a-service. Realms provides DAO-as-a-service. Each demonstrates that a single audited factory can serve a heterogeneous set of project needs at low marginal cost. REDAX extends this pattern into the merger domain.

**Sol-Incinerator.** A Solana wallet cleanup tool that lets users dispose of dust tokens and recover rent SOL. Operates on the user-side of the dead-token problem. REDAX operates on the project-side: instead of burning dust for rent, REDAX helps the project bring its community forward into a new token.


## Team

**Tuncer Timur** — Co-founder, Technical. Architecture, smart contract development, security. Leads the Rust and Anchor work.

**Ahmet Akın Aktan** — Co-founder, BD. Business development, partnerships, communications. Leads ecosystem outreach and campaign creator relations.

We are a focused founding team and are actively expanding our advisor network and technical bandwidth. Strategic advisory conversations are in progress; names will be added here only after explicit approval. Solana ecosystem outreach is also in progress.

A boutique Solana audit firm will be selected for the Phase 1 boutique audit from the candidate set described in the Security section. Premier-firm engagement is targeted for Phase 2 and beyond.


## Roadmap

| Quarter | Milestone |
|---|---|
| Q2 2026 | SPEC v1 finalization · Devnet deployment · Public testnet preparation |
| Q3 2026 | Tier 2 audit · Tier 3 audit · Solana Foundation grant outreach · Guarded mainnet preparation |
| Q4 2026 | Phase 2 features · Verified tier · Solana Frontier Hackathon submission |
| Q4 2026 (late) | Tier 4 audit · Bug bounty live · Open mainnet |
| 2027 | Phase 3 (subject to legal clearance) · Multi-campaign discovery UI · Custom indexer |

These dates are targets, not commitments. We will delay any milestone if quality requirements are not met. We would rather ship a slow, working protocol than a fast, fragile one.


## Current Status

As of this document's date:

- SPEC v1 is being finalized. The architecture, the math, the security boundaries, and the phased rollout plan are settled at the design level.
- The protocol has not yet been deployed. We are about to begin the Rust implementation.
- We are not in an active fundraising round. We are focused on the product and the early-stage community.
- We are open to advisor conversations, partnership conversations, and feedback from anyone in the Solana ecosystem who has thought about this problem space.

This document will be revised as the protocol develops. Each version will be timestamped and previous versions will remain available in the public repository.


## Contact

**Web** — https://redaxhub.com  
**Documentation** — https://docs.redaxhub.com  
**GitHub** — https://github.com/redaxhub  
**X (Twitter)** — [@redaxhub](https://x.com/redaxhub)  
**Telegram** — https://t.me/redaxhub  

**Email** — hello@redaxhub.com (activation in progress)

This document has been prepared for community review and advisory feedback. It is not a fundraising prospectus, an investment offering, or a commitment to launch any token. It describes a protocol under development.


## References

This Litepaper draws on public sources for its quantitative claims and prior-art discussion. Categories of source material:

- **Pump.fun scale and graduation rate**: DWF Labs research on Pump.fun token launch volumes, Dune Analytics dashboards tracking graduation rates, and reporting from The Defiant and Cointelegraph on graduation-rate trends in 2025.
- **ASI Alliance migration**: Fetch.ai official blog announcement of conversion rates and FDV, Ocean Protocol blog post on the final ASI structure.
- **Pak's Merge**: Guinness World Records entry for Most Expensive NFT Artwork (Open-Edition Auction), and PR coverage of the December 2021 Nifty Gateway sale.
- **Ribbon to Aevo**: Coinbase asset page and Ribbon Finance DAO governance archive.
- **Solana Token-2022 and execution stack**: Solana official documentation on Token Extensions, priority fee markets, and compute budget; Helius technical writeups on local fee markets.
- **Reference implementations cited in Precision Sandwich**: Uniswap V3 Core repository, OpenZeppelin Contracts, EIP-4626 specification at eips.ethereum.org.
- **Existing Solana factories**: Official documentation and product pages of Streamflow, Squads Protocol, and Realms.

A more complete bibliography with direct links will accompany the full technical documentation when published.
