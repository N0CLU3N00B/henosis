# Henosis

**There is no token. Ever.** No token sale, no airdrop, no governance token, no fee token. Voting weight belongs
to verified persons, never to coins.

Henosis is an open, permissionless governance layer for [Shibarium](https://shibarium.shib.io) (Chain 109). It lets
communities at the local, regional and global level:

- verify people uniquely and with as little data as possible: one human, one identity;
- record mutual aid as non-transferable karma;
- settle allocations that cannot be decided fairly otherwise by a verifiable lottery;
- vote secretly and resistant to manipulation and coercion;
- decide at the level that is actually responsible (subsidiarity);
- use AI for advice only, never for decisions.

Henosis grew out of the Shib community and was first conceived as the governance layer for SHIB OS (working title
"SHIB OS – Unity Layer").

This repository holds the public documentation only. It contains no code.

## Contents

| File | What it is |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Architecture overview: layers, modules M1 to M8, threat model |
| [INTERFACES.md](INTERFACES.md) | The three neutral interfaces for scoped proof of personhood, described as text |
| [ROADMAP.md](ROADMAP.md) | Milestones to Release 1 |
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

## Status (October 2026)

| Module | State |
|---|---|
| M1 Identity (electronic passports and ID cards) | Built: zero-knowledge registration for the common passport signature schemes, chip authentication upgrades, certificate registries with time lock and objection, admission of new document signer certificates by proof. Phone app in development |
| M4 Voting, stage 1 (Scope 0, non-binding) | Built: anonymous membership per epoch, proposals and revoting; a cleartext-tally variant and an FHE-encrypted variant (Zama). Next: voting v2 with a commit-reveal secret ballot ([roadmap](ROADMAP.md)) |
| M5 Scope registry | Prototype built |
| M2 Karma, M3 Lottery, M6 Constitution, M7 AI advisory, M8 Operations | Design only, no code |

- **Demonstrated on a testnet only.** The full flow (register, join, propose, vote, tally) ran on Celo Sepolia
  with synthetic documents.
- **Not ready for real use:**
  - all proving keys so far come from local single-party setups, so the demo is not tamper-proof; a multi-party
    ceremony is still needed;
  - no external audit yet;
  - no relayer and no released phone app.
- **Nothing is deployed on Shibarium.** Mainnet only after an external audit.
- **First use case:** the Shiba Network State as Scope 0, a person-based shadow vote next to the existing
  token-weighted Shiba Inu DAO, non-binding until the community chooses otherwise.

## License

- Documentation in this repository: [Apache-2.0](LICENSE).
- The ERC draft in [`erc/`](erc/): CC0 1.0, as the ERC process requires ([`LICENSE-CC0`](LICENSE-CC0)).
