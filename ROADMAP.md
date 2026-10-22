# The way to Release 1

Release 1 is the core: one verified human, one vote. The steps below are in order; each starts when the one before
it is done. The target for Release 1 is **early 2027**. That is a target, not a promise: an audit or a ceremony that
finds something moves it.

| Step | Content | State |
|---|---|---|
| Built | Identity with a nullifier bound to chain and registry (v2), voting v2 with a commit-reveal secret ballot and private join, reference relayer, draft ERC | done, tested locally and in CI, not deployed |
| Hardening | Fixed freeze criteria, independent rebuild of the circuits, audit package | now |
| Freeze | The circuits are frozen first, the contracts after the ceremony, because the verification keys change the contracts' addresses | next |
| External audit | The frozen circuits, then the contracts with their verifier integration; remediation after each part | after the freeze |
| Multi-party ceremony | Trusted setup for the frozen circuits with several independent participants and a public transcript | after the circuit audit |
| Release 1 | Deployment of Scope 0 on Shibarium mainnet, its home chain; the ERC submitted; the code public | target early 2027 |

## What Release 1 contains

- **An audited core:** registration, certificate admission, nullifier v2, and voting v2 with child scopes and
  admission rules. The core is frozen before the audit.
- **Experimental satellites** next to the core, not frozen. The first one is the **Epoch badge register**:
  - a record per epoch, kept by a contract, of addresses whose holders chose to record their membership;
  - obtained through a zero-knowledge proof against a snapshot;
  - no token standard and nothing transferable;
  - no rule may read it as a threshold.

## Deferred until after Release 1

These modules stay in the design ([ARCHITECTURE.md](ARCHITECTURE.md)). They were deferred on 13 October 2026, when
the work concentrated on the core; none of them is dropped.

- **Karma** (module M2), **lottery** (M3), **constitution** (M6) and **AI advisory** (M7): design only today.
- **Encrypted tally (FHE)** with Zama, through the vote-format slot; Release 1 uses a commit-reveal secret ballot.
- **STARK-based and EIP-8288 verifiers**, through the governed verifier slots.
- **Further admission paths** without an electronic document: local vouching and presence events.
- **Extension ERCs** (proof-verifier interface, vote formats, pseudonym attestations) after the core ERC is submitted.
