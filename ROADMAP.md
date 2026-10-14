# Roadmap to Release 1

Milestones M1 to M6 below are steps on the way to the first release. They are not the architecture modules M1 to M8
of [ARCHITECTURE.md](ARCHITECTURE.md).

| Milestone | Content | Target |
|---|---|---|
| M1 | Chain-scoped nullifier (v2), all instances re-keyed, blind review | Nov 2026 |
| M2 | Voting v2: commit-reveal secret ballot, abstention, enforced quorum, rules changeable by vote, pluggable vote format, private join | Nov 2026 |
| M3 | Reference relayer, key hosting, app end-to-end on testnet | Nov–Dec 2026 |
| M4 | Freeze, audit package, audits start | Dec 2026 |
| M5 | Audit remediation, trusted-setup ceremony | Jan–Feb 2027 |
| M6 | Release 1 on Shibarium mainnet, ERC submitted, code public | Feb–Mar 2027 |

## What Release 1 contains

- **An audited core:** registration, certificate admission, nullifier v2, and voting v2 with child scopes and
  admission rules. The core is frozen at M4.
- **Experimental satellites** next to the core, not frozen. The first one is the **Epoch badge register**:
  - a record per epoch, kept by a contract, of addresses whose holders chose to record their membership;
  - obtained through a zero-knowledge proof against a snapshot;
  - no token standard and nothing transferable;
  - no rule may read it as a threshold.

## After Release 1

- **Encrypted tally (FHE)** once capacity allows; Release 1 uses a commit-reveal secret ballot.
- **Karma (M2 of the architecture), lottery (M3), constitution (M6), AI advisory (M7):** design only today; they
  follow after Release 1.
