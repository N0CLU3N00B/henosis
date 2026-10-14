---
eip: TBD
title: Scoped Proof of Personhood
description: Interfaces to verify one-person membership per scope, record scoped nullifiers, and plug in admission rules
author: TBD (the project owner decides authorship before any submission)
discussions-to: TBD
status: Draft (not submitted)
type: Standards Track
category: ERC
created: 2026-10-13
requires: 165
---

## Abstract

This standard defines three small interfaces for applications that need "one person, one action" within a scope,
without learning who the person is:
- **`IPersonhoodVerifier`** checks a proof that the prover belongs to a personhood set (one entry per person). It
  outputs the prover's nullifier for one scope, bound to one message.
- **`INullifierRegistry`** records used nullifiers per consumer and context.
- **`IAdmissionRule`** expresses one further condition a scope places on its members (holding an asset, an
  attestation, vouches, presence), as a binary check that never adds weight.

The interfaces do not depend on the proof system, the kind of credential that put the person into the set (a
government ID, a presence event, a vouching scheme), or the chain.

## Motivation

Many projects implement the same three pieces with incompatible signatures:
- a membership proof against a set of persons;
- a nullifier to stop double use;
- extra conditions for a community ("holders only", "vouched members only").

Examples are identity registries built on electronic passports, Semaphore groups and anonymous voting contracts.
Because each piece is tied to one proof system or one credential type:
- a community cannot swap a passport-based personhood set for a presence-based one;
- it cannot move from one proof system to another (for example to a post-quantum one);
- it cannot reuse an admission rule written for another project.

Wallets and indexers cannot recognise any of these pieces either.

This standard fixes only the seams:
- what a personhood proof states, and how its nullifier is scoped;
- how used nullifiers are recorded without one consumer interfering with another;
- how a scope's extra conditions are checked and described.

Everything else stays with the scope: who may join, thresholds, councils, constitutions. Weighting by holdings is
explicitly out of scope: an admission rule admits or refuses, it never weighs.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT
RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Definitions

- **Personhood set:** a set of identity commitments, at most one per person, committed to by a root (`setRoot`). How
  persons enter it is outside this standard.
- **Scope:** a field element naming the domain in which a person may act once. Examples: one community, one proposal,
  one airdrop.
- **Nullifier:** a deterministic function of the person's identity secret and the scope. It is unique per (person,
  scope) and unlinkable across scopes without the secret.
- **Message:** a field element the proof is bound to (an action hash, a vote option).
- **Context:** a `bytes32` chosen by a consumer of nullifiers (a registration, an epoch, a proposal).
- **Pseudonym:** the person's nullifier in a scope, used by admission rules as the person's identifier in that scope.

### `IPersonhoodVerifier`

```solidity
interface IPersonhoodVerifier is IERC165 {
    error MalformedProof();
    error ValueOutOfField(uint256 value);
    event PersonhoodVerifierDeployed(bytes32 indexed proofSystem, bytes32 indexed nullifierScheme, uint256 maxSetSize);
    function verifyPersonhood(uint256 setRoot, uint256 scope, uint256 message, uint256 nullifier, bytes calldata proof)
        external view returns (bool valid);
    function personhoodScheme() external view returns (bytes32 proofSystem, bytes32 nullifierScheme);
}
```

- `verifyPersonhood` MUST return true only if the proof shows all three of:
  1. the prover knows the secret of an identity in the set committed to by `setRoot`;
  2. `nullifier` is the implementation's nullifier of that identity for `scope`;
  3. the proof is bound to `message`.
- It MAY revert with `MalformedProof` or `ValueOutOfField` instead of returning false.
- It MUST NOT reduce out-of-field values silently.
- It MUST NOT keep state, so the same proof verifies the same way every time. Use-once is the caller's
  (`INullifierRegistry`), and so is the choice of acceptable `setRoot` values.
- The proof is opaque bytes of any length. The function MAY call other contracts. Callers MUST NOT assume a fixed proof
  size or gas budget.
- `personhoodScheme` names the proof system and the nullifier derivation, so clients can produce matching proofs.
- `PersonhoodVerifierDeployed` SHOULD be emitted once at deployment.

### `INullifierRegistry`

```solidity
interface INullifierRegistry is IERC165 {
    error NullifierAlreadyConsumed(address consumer, bytes32 context, uint256 nullifier);
    event NullifierConsumed(address indexed consumer, bytes32 indexed context, uint256 indexed nullifier);
    function consume(bytes32 context, uint256 nullifier) external;
    function isConsumed(address consumer, bytes32 context, uint256 nullifier) external view returns (bool consumed);
}
```

- `consume` MUST key the entry by `(msg.sender, context, nullifier)` and MUST revert with `NullifierAlreadyConsumed`
  if it exists. A consumer can therefore only write in its own namespace.
- Entries MUST NOT be removable.
- A consumer that lets a person change an earlier action (a revote) MUST NOT consume per change. It consumes once,
  or keeps its own versioning.

### `IAdmissionRule`

```solidity
interface IAdmissionRule is IERC165 {
    enum Phase { Join, Epoch, Action }
    error NotAdmitted(bytes32 kind, uint8 reason);
    event AdmissionRuleConfigured(bytes32 indexed kind, uint16 version, bytes params);
    function check(uint256 scope, uint256 pseudonym, Phase phase, bytes32 context, bytes calldata evidence)
        external view returns (bytes32 ruleNullifier);
    function describe() external view returns (bytes32 kind, uint16 version, bytes memory params);
}
```

- `check` MUST revert with `NotAdmitted` when the person is not admitted. It MUST NOT change state.
- When one piece of evidence must not admit two persons in the same context (one NFT, one wallet), the rule returns a
  non-zero `ruleNullifier` derived from that evidence and `context`, and the caller MUST record it as used. Otherwise
  the rule returns 0.
- A caller that promises anonymous actions MUST use only zero-knowledge evidence at `Phase.Action`. Public evidence at
  that phase links the action to the evidence.
- `describe` returns the rule kind, the version of its evidence and parameter format, and its parameters. Clients
  SHOULD show them to a person before the person joins.
- A rule MUST NOT return or imply a weight.

### ERC-165 identifiers

| Interface | ID |
|---|---|
| `IPersonhoodVerifier` | `0x461ae3b6` |
| `INullifierRegistry` | `0xcafeb4e0` |
| `IAdmissionRule` | `0x4d42cf5d` |

Each identifier is the XOR of the interface's own function selectors (`Phase` is encoded as `uint8`):
`0x12724690 ^ 0x5468a526`, `0x98017489 ^ 0x52ffc069` and `0x57934ba2 ^ 0x1ad184ff`.

Implementations MUST return true from `supportsInterface` for their interface's identifier and for ERC-165's.

## Rationale

- **Field elements, opaque proofs.** The scope, message, nullifier and root are field elements, because every SNARK
  nullifier scheme needs them; the proof stays bytes. A hash-based (post-quantum) verifier can implement the same
  interface. Its proofs are tens to hundreds of kilobytes and need several contracts, so no size or gas bound is
  fixed.
- **Stateless verifier, separate registry.** Keeping use-once out of the verifier lets one verifier serve every scope.
  A consumer can also choose its contexts (per epoch, per proposal) without a new verifier.
- **Namespacing by `msg.sender`.** As in ERC-7812's key isolation, no consumer can mark another's nullifiers. Without
  it, a shared registry would let anyone block a nullifier in someone else's context.
- **Rules return a nullifier instead of keeping state.** A rule deployment is shared by many scopes; the caller
  decides where the rule nullifier is recorded.
- **Phases.** When a condition is checked decides what it leaks. A check per action is only private as a
  zero-knowledge proof. The phase is a parameter of the scope, not of the rule.
- **No weight.** "One person, one vote" is the purpose of the standard. Token-weighted voting has its own standards
  (ERC-1202, governor contracts).

## Related Work

- **ERC-7812 (ZK Identity Registry, Review):**
  - a singleton `EvidenceRegistry` stores abstract statements as key/value pairs in a sparse Merkle tree (Poseidon,
    height 80 in the reference);
  - keys are isolated per registrar (`hash(msg.sender, key)`), and registrars define their own statements and
    circuits.
  - It is complementary. A personhood set's root, a scope's snapshot root or a certificate root can be **stored as an
    ERC-7812 statement** by a registrar, so other applications can prove facts about it against one global root.
  - This standard covers what ERC-7812 leaves to registrars: the personhood statement, scoped nullifiers and
    admission rules.
  - We follow its isolation rule (`INullifierRegistry` namespaces) and its advice to pass a root as a parameter and
    check it against the registry's history, instead of reading the current root on chain.
- **ERC-1202 (Voting Standard, Review):**
  - its votes carry a `weight` (`eligibleVotingWeight(proposalId, voter)`), its events index the **voter address**,
    and it assumes one address casts one decision.
  - Each of these contradicts the goal here: no weight, no address, one person instead of one address.
  - It is not built on. A personhood-based voting contract could still expose ERC-1202-style views with weight 1, but
    never the voter.
- **ERC-7524 (PLUME, Draft):**
  - a deterministic signature on secp256k1 whose output `nul = h^sk` (with `h` hashed from the message and the public
    key) is a nullifier for an Ethereum account, provable in a SNARK without revealing the account.
  - It fits **`IAdmissionRule` as a holder rule**: "this person controls an address in the holder snapshot", with the
    PLUME nullifier for (address, context) as the `ruleNullifier`, without linking the wallet to the person.
  - It does not replace personhood: one person can hold many addresses.
  - PLUME notes that a quantum adversary who breaks ECDSA could recompute nullifiers; the same applies to such a rule.
- **Semaphore (v4):**
  - groups with an admin, `addMember`, and `validateProof(groupId, proof)`, where the proof carries `merkleTreeRoot`,
    `nullifier`, `message` and `scope`.
  - `IPersonhoodVerifier` generalises its verification: a set root instead of a group id, the scheme named, the proof
    opaque.
  - A Semaphore group whose members are one per person is a personhood set, and an adapter over Semaphore's verifier
    implements `IPersonhoodVerifier`.
  - Semaphore's own nullifier check (`validateProof` stores it) corresponds to `INullifierRegistry` with the group as
    the namespace.

## Backwards Compatibility

New interfaces only. Existing personhood and voting contracts can expose them through adapters without redeployment.

## Reference Implementation

Henosis, a governance layer on Shibarium, maps its existing contracts to these interfaces: its anonymous voting
verifies membership in a per-epoch group filled from its identity registry (the role of `IPersonhoodVerifier`), and
records registrations, joins and proposals once per nullifier (the role of `INullifierRegistry`); votes are
versioned to allow revoting. Admission rules (`IAdmissionRule`) are planned there.

## Security Considerations

- **The set decides Sybil resistance.**
  - This standard proves membership in a set, not that the set has one entry per person. A consumer accepts only
    `setRoot` values from a source it trusts, and checks them against that source's root history.
- **Scope collisions across chains and contracts.**
  - A nullifier is unique per scope. Two applications that use the same scope value link a person's actions across
    them, and can block each other.
  - Scopes SHOULD include the chain id and the consuming contract's address, for example
    `scope = hash(chainId, consumer, localScope)` reduced into the field.
- **Message binding.** Without a message bound in the proof, a proof can be front-run and reused for another action.
  Consumers MUST compute `message` from the action themselves, never accept it from the caller.
- **Linkage through admission rules.**
  - Public evidence at `Phase.Action` links actions to evidence.
  - The set of attestations or rule nullifiers at a pseudonym can be a fingerprint.
  - Clients SHOULD show this before joining.
- **Field overflow.** A value at or above the field modulus, accepted and reduced, gives a second encoding of the same
  nullifier. Verifiers MUST refuse it (`ValueOutOfField`), and registries SHOULD as well.
- **Verifier replacement.**
  - A consumer that can swap its `IPersonhoodVerifier` can swap its security. Replacement SHOULD go through a time
    lock with an objection window, and SHOULD apply only from an epoch boundary.
- **Deterministic nullifiers after a key compromise.**
  - Whoever learns an identity secret can compute all of that person's nullifiers, past and future, and link them.
  - The same holds for PLUME-style nullifiers under a quantum adversary.
- **Weight.** Treating a rule's evidence (a balance) as weight breaks the purpose of this standard. Rules MUST NOT
  expose a weight, and consumers MUST NOT derive one from them.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE-CC0).
