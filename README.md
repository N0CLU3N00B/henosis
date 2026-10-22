# Henosis

**There is no token. Ever.** No token sale, no airdrop, no governance token, no fee token. Voting weight belongs
to verified persons, never to coins.

Henosis is an open, permissionless primitive for **one verified human, one vote**. Communities, DAOs and games use
it so that every member acts once per decision, and nobody learns who the member is: the proof shows that a real
person takes part, not which one. It lets a community:

- verify people uniquely and with as little data as possible: one human, one identity, from the chip of a passport
  or ID card, proven in zero knowledge on the phone;
- vote secretly, once per person and proposal, with a commit-reveal secret ballot;
- set its own admission rules and procedures behind fixed interfaces, within four guarantees that cannot be
  configured away (uniqueness per scope, anonymity of the vote, unilateral exit, decisions only about oneself);
- decide at the level that is actually responsible (subsidiarity).

Since 13 October 2026 the work concentrates on this core. Karma, the verifiable lottery, the constitution and the AI
advisory layer stay in the design and are **deferred until after Release 1** ([ARCHITECTURE.md](ARCHITECTURE.md)).

Henosis grew out of the Shib community and was first conceived as the governance layer for SHIB OS (working title
"SHIB OS – Unity Layer").

## Any EVM chain

- **Henosis can be deployed on any EVM chain.** Its interfaces are chain-neutral and proposed as a draft ERC.
- **Shibarium (Chain 109) is the home chain of Scope 0,** the first global scope. Every decision has one home chain;
  other chains only read its result.
- **Nullifiers are separated per chain.** The same document gives a different nullifier on every chain and
  registry, so registrations cannot be linked across chains.
- **Other chains read results by state proofs,** never by asset bridges or relayers with a veto.
- **Encrypted tallying and new proof systems plug in through slots.** Zama's fully homomorphic encryption (FHE)
  comes in through the vote-format slot; STARK-based verifiers and verifiers built on EIP-8288 come in through the
  governed verifier slots. The core is built for both before either arrives ("pre-Zama, pre-STARK"); neither is
  part of Release 1.

This repository holds the public documentation only. It contains no code.

## Contents

| File | What it is |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Architecture overview: layers, modules M1 to M8, threat model |
| [INTERFACES.md](INTERFACES.md) | The three neutral interfaces for scoped proof of personhood, described as text |
| [ROADMAP.md](ROADMAP.md) | The way to Release 1 |
| [erc/scoped-proof-of-personhood.md](erc/scoped-proof-of-personhood.md) | Draft ERC "Scoped Proof of Personhood" (not submitted) |

## Principles

| # | Principle | What it means for the design |
|---|---|---|
| P1 | One human, one vote | Voting weight is tied to a verified person, never to token ownership |
| P2 | Influence is not for sale | Karma and lottery tickets are soulbound; there are no transferable governance tokens |
| P3 | Subsidiarity | Every decision sits at the lowest level that can resolve it |
| P4 | Homeostasis instead of maximisation | Caps, decay and saturation are mandatory parameters of every module |
| P5 | Decentralisation and redundancy | No single keys, no central authority, several independent operators |
| P6 | Privacy by default | Only proofs and nullifiers go on chain, never personal data in plain text |
| P7 | Constitution before majority | Fundamental rights are not subject to a vote |
| P8 | AI advises, never decides | AI components hold no keys and have no execution rights |
| P9 | Shibarium, SHIB OS and Zama first | Every technology choice checks a Shibarium-, SHIB OS- or Zama-based solution first |
| P10 | Whoever bears the risk decides | A risk that falls on one person is that person's choice; a risk that falls on everyone is decided by binding governance |
| P11 | No home-made cryptography | Only standardised or publicly reviewed building blocks, combined, implemented and audited externally |

Further fixed rules:

- **No admin keys, no owner** in any contract. Upgrades only through the constitutional procedure (M6).
- **No token-weighted voting**, no staking or validator coupling, no burn logic or fee hooks in the protocol.
- **No bridges as a required component.** Results are read across chains by state proofs, not by asset bridges or
  relayers with a veto.

## Status (22 October 2026)

| Part | State |
|---|---|
| M1 Identity (electronic passports and ID cards) | Built: zero-knowledge registration from the document chip for the common passport signature schemes, with a nullifier bound to chain and registry (v2); chip authentication upgrades; certificate registries with time lock and objection; admission of new document signer certificates by proof. Phone app in development |
| M4 Voting v2 | Built: commit-reveal secret ballot, abstention, enforced quorum, private join, rules changeable by vote, a vote-format slot and an admission-rule slot. It replaces stage 1 (anonymous membership with revoting, in a cleartext and an FHE variant) |
| M5 Scope registry | Prototype: nested scopes, child scopes and dissolution |
| Reference relayer | Built: submits transactions for users, so nobody needs gas or crypto to take part; finality-aware and restart-safe. Not operated publicly |
| Draft ERC | "Scoped Proof of Personhood": draft, not submitted ([erc/](erc/scoped-proof-of-personhood.md)) |
| M2 Karma, M3 Lottery, M6 Constitution, M7 AI advisory | Deferred until after Release 1: design only, no code |

- **Demonstrated on a testnet only.** The full flow (register, join, propose, vote, tally) ran on Celo Sepolia
  with synthetic documents, with voting stage 1 and the circuits of 9 October 2026. Nullifier v2, voting v2 and the
  relayer are tested locally and in CI, not deployed.
- **Not ready for real use:**
  - all proving keys so far come from local single-party setups, so the demo is not tamper-proof; a multi-party
    ceremony is still needed;
  - no external audit yet;
  - the relayer is not operated publicly and the phone app is not released.
- **Nothing is deployed on Shibarium mainnet (Chain 109)**, and nothing on the Puppynet testnet (Chain 157). The
  only deployment so far is the demo on Celo Sepolia. Mainnet only after an external audit.
- First use case: a non-binding, person-based shadow vote for the Shib community (Scope 0), until the community
  chooses otherwise.

## License

- Documentation in this repository: [Apache-2.0](LICENSE).
- The ERC draft in [`erc/`](erc/): CC0 1.0, as the ERC process requires ([`LICENSE-CC0`](LICENSE-CC0)).
