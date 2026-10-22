# Henosis architecture (overview)

A shortened overview of the architecture specification. It describes the design and the current approach of each
module.

Release 1 is the core: identity (M1), voting (M4) and the scope registry (M5), with the security and operations work
of M8 they need. Karma (M2), the lottery (M3), the constitution (M6) and the AI advisory layer (M7) stay in the
design and are **deferred until after Release 1** (decided 13 October 2026, when the work concentrated on the
primitive "one verified human, one vote"). Henosis can be deployed on any EVM chain; Shibarium is the home chain of
Scope 0.

## 1. Layer model

```
┌──────────────────────────────────────────────────────────────┐
│  Clients: phone / web / offline node                           │
│  SDK · wallet · relayer · local proof generation               │
└───────────────┬──────────────────────────────────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────┐
│  M7  AI advisory layer (off-chain, plural, no keys) [deferred] │
└───────────────┬──────────────────────────────────────────────┘
                │ hashes of the advisory documents
┌───────────────▼──────────────────────────────────────────────┐
│  M6  Constitution (rights, amendments, brake)      [deferred]  │
├──────────────────────────────────────────────────────────────┤
│  M5  Scope registry (local → regional → global)                │
├──────────────┬──────────────┬──────────────┬─────────────────┤
│  M1 Identity │  M2 Karma    │  M3 Lottery  │  M4 Voting      │
│  (PoP, ZK)   │  [deferred]  │  [deferred]  │  (anonymous)    │
└──────────────┴──────────────┴──────────────┴─────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────┐
│  M8  Security and operations (keys, audits, post-quantum)      │
├──────────────────────────────────────────────────────────────┤
│  Any EVM chain · home chain of Scope 0: Shibarium (Chain 109)  │
└──────────────────────────────────────────────────────────────┘
```

## 2. Non-goals

- No own token, no speculation, no tokenomics.
- No coupling of governance to staking or to Shibarium's validator consensus.
- No central identity database.
- No bridges as a required component.
- No burn logic, no binding to SHIB, no fee hooks in the protocol. Shibarium benefits through use and as a showcase.
  Revenue, if there ever is any, comes only from a separate legal entity offering services, never from the protocol.

## 3. Modules

### M1 – Identity (proof of personhood)

**Goal:** prove "a unique human, member of scope X" without revealing who.

- **Electronic identity documents (built; phone app in development).** The person's phone reads the chip of a
  passport or ID card over NFC and proves in zero knowledge, on the phone:
  - the document was signed by a document signer certificate (DSC) of an issuing state;
  - that certificate is in a public tree of known certificates;
  - the person holds a secret bound to the document.

  On chain there is only the proof, a document nullifier (one registration per document and registry; it differs
  between registries, so registrations cannot be linked across chains) and an identity commitment. No name, number or date of birth leaves the phone.
- **Certificate trees.** The country signing certificates (CSCAs) and the DSCs live in public Merkle trees.
  - New roots are proposed with a deposit, wait out a time lock and can be stopped by an objector during that time.
  - New DSCs can also be admitted permissionlessly by a zero-knowledge proof that a known CSCA signed them, again
    with a time lock and an objection window.
- **Chip authentication.** Documents whose chip can sign a challenge can upgrade an identity to a higher assurance
  level, which shows that the chip itself was present and not a copy of its data.
- **Further admission paths (designed, deferred until after Release 1):** local vouching by verified members, and
  presence events with rotating, randomly selected verifiers for people without an electronic document.
- **Recovery.** Rebinding an identity to a new secret by the chip or by presence, with a waiting period and an
  objection, and a successor path for a new document.

### M2 – Karma (soulbound) — deferred until after Release 1

**Goal:** record mutual aid and community service, non-transferable.

- Granted only on mutual confirmation (the helper reports, the recipient confirms) or by scope services.
- Caps per pair of persons and epoch and per person and epoch; karma decays over time.
- Reciprocity loops are detected off-chain and reviewed by randomly selected reviewers.
- Externally only threshold proofs ("at least X karma"), no individual history.
- Karma converts into lottery tickets or exempts from a duty. It cannot be transferred, sold or delegated.

### M3 – Lottery — deferred until after Release 1

**Goal:** allocations that cannot be decided fairly otherwise: time slots, offices between equally qualified people,
scarce goods.

- One base entry per person (by nullifier), extra tickets from karma.
- Public, verifiable randomness (a beacon such as drand, or a VRF), fixed before the participation deadline.
- Sortition fills offices (reviewers, the emergency brake body of M6) from volunteers for a fixed term.

### M4 – Voting

**Goal:** secret, verifiable, resistant to vote buying and coercion.

- **Stage 1 (built):** a non-binding shadow vote in Scope 0.
  - Members join an epoch group with their identity and vote with a group membership proof. The proof shows that the
    voter is a member, not which one.
  - One vote nullifier per member and proposal stops double voting; a member may change their vote until the
    window closes.
  - **What is public, in both variants:**
    - who joined the epoch;
    - that a vote nullifier voted, and how often (each revote is visible as such);
    - the result.
  - **What differs between the two variants:**
    - **cleartext tally**, for chains without fully homomorphic encryption: each vote's content is public, tied to
      its vote nullifier, and the running tally can be read while voting is open. Anonymous, but not a secret
      ballot.
    - **FHE tally** with Zama, for chains where Zama runs: each vote and the running tally stay encrypted; only the
      final sums are decrypted after the window closes.
  - Whoever was shown a voter's vote nullifier can see whether that voter later voted again. With the cleartext tally
    they can also see the new choice.
- **Voting v2 (Release 1, built):** a commit-reveal secret ballot, abstention, an enforced quorum, rules changeable
  by vote, pluggable vote formats and a private join. A second commit inside the commit window overwrites the first.
  The FHE tally with Zama comes in through the vote-format slot after Release 1.
- **Target design (later stage):** the MACI principle. Every person can change their voting key without anyone
  noticing, so a bought or coerced vote cannot be proven. The coordinator role is spread over several independent
  operators, and the tally is proven in zero knowledge.
- **Delegation (planned):** topic-specific and revocable at any time, with a cap per delegate; a direct vote always
  overrides the delegation.

### M5 – Scope registry (federation)

**Goal:** represent subsidiarity technically.

- A scope has a parent (local → regional → global), an identity group, parameters (quorums, deadlines, caps) and a
  list of competences.
- A scope decides only within its competences.
- Competences move only upwards, ceded from below, can be reclaimed from below, and can never be taken from above.
- **Interfaces instead of rules:**
  - Henosis supplies interfaces and building blocks; each community sets its own rules, thresholds and procedures.
    Admission rules (holding an asset, an attestation, vouches, presence) plug in behind fixed interfaces
    ([INTERFACES.md](INTERFACES.md)). A rule admits or refuses; it never adds weight.
  - Four guarantees hold for every scope and cannot be configured away:
    - **uniqueness per scope:** one person, one membership, one vote per proposal;
    - **anonymity of the vote;**
    - **unilateral exit:** leaving needs no deposit, lock or approval;
    - **decisions only about oneself:** a scope acts only in its own matters, never in another scope.
  - Creating child scopes and dissolving a scope are built in the prototype.
- **Chains:**
  - scopes are contracts on any EVM chain; Scope 0 lives on Shibarium, its home chain;
  - every decision has one home chain; other chains read its result by state proofs, never by bridges;
  - nullifiers are separated per chain and registry, so a person's registrations cannot be linked across chains;
  - large scopes can later run on chains of their own with the same protocol.

### M6 – Constitution — deferred until after Release 1

- **Fundamental rights,** immutable in the protocol: participation, exit, privacy, equality of the vote.
- **Amendments** to other rules need a supermajority across several scopes and a time lock.
- **Emergency brake:**
  - a randomly selected, time-limited body can pause single modules;
  - it can never move funds or change rules;
  - every pause expires by itself and needs confirmation by a vote.
- **No admin keys.** Until M6 exists, an interim objector can only object (veto during a time lock, a time-limited
  pause). The role is meant to pass to M6, acting through signatures collected off-chain.

### M7 – AI advisory layer — deferred until after Release 1

- At least three independent models from different operators, at least one with open weights.
- Each side of a decision gets its own argument (the "voting booklet" principle); facts with sources are kept
  apart from evaluation.
- Prompts, model versions and outputs are hashed and referenced on chain.
- No keys, no execution rights, no payment access.

### M8 – Security and operations

- Threshold signatures or MPC for every operator role.
- External audits and formal verification of all contracts before mainnet.
- Upgrades only through the M6 procedure with a time lock.
- Crypto-agility: signature and proof systems are swappable through governed verifier slots (time lock and
  objection), so STARK-based verifiers, verifiers built on EIP-8288 and post-quantum schemes can be adopted when
  standards and audited implementations exist. The Release-1 core stays on Groth16 over BN254.
- Gas abstraction: users need no gas token (BONE on Shibarium); transactions are submitted by a relayer (with a
  direct-send fallback), funded from a community pool. A reference relayer is built. Account abstraction is
  revisited once an ERC-4337 EntryPoint exists on Shibarium.
- Offline-first clients: proofs are generated on the device, submitted through a relayer or a local node.

## 4. Example: a vote in a local scope

The example shows the full design. In Release 1, step 2 (M7) and the emergency brake of step 6 (M6) are not yet
there.

1. A member creates a motion; its hash goes on chain.
2. M7 prepares arguments for and against from several models; hashes on chain, content on distributed storage.
3. Members vote with a membership proof (M1) and a commitment to their vote (M4).
4. A vote can be changed until the commit window closes; then the votes are revealed.
5. The tally is computed and published with its proof.
6. The result takes effect after a time lock, unless the emergency brake (M6) is invoked.

## 5. Threat model

| Threat | Countermeasure |
|---|---|
| Duplicate identities (Sybil) | M1: one registration per document, nullifiers, vouching with liability |
| Karma farming by collusion | M2: pair caps, decay, graph analysis, randomly selected reviewers |
| Vote buying and coercion | M4: revoting (stage 1), unprovable key changes (target design) |
| Flash-loan takeover | Excluded by design: no token-weighted governance |
| Coordinator compromise | Several independent operators; the tally is proven |
| Manipulation through AI preparation | M7: plurality, partisan preparation, hashing |
| Admin key abuse | No admin keys; upgrades only through the constitution |
| Overreach by a higher level | M5: competences are only ceded from below |
| Quantum computers | M8: crypto-agility, post-quantum roadmap |
