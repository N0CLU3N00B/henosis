---
eip: TBD
title: Scoped Proof of Personhood
description: Interfaces to verify one-person membership per scope, record scoped nullifiers, and plug in admission rules
author: Henosis (the GitHub handle is set at submission)
discussions-to: TBD (an Ethereum Magicians thread, opened at submission)
status: Draft
type: Standards Track
category: ERC
created: 2026-10-13
requires: 165
---

> **Not submitted.** This draft is published for comment. The reference implementation and its tests will be placed
> under `assets/` when it is submitted.

## Abstract

This standard defines three small interfaces for applications that need "one person, one action" within a scope,
without learning who the person is:

- `IPersonhoodVerifier` checks a proof that the prover belongs to a personhood set (one entry per person). It
  outputs the prover's nullifier for one scope, bound to one message.
- `INullifierRegistry` records used nullifiers per consumer and context.
- `IAdmissionRule` expresses one further condition a scope places on its members (holding an asset, an
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
  persons enter it is outside this standard. The set's own **entry scheme** (for example the document nullifier that
  keeps one entry per person) is a property of the set, not of the verifier; `personhoodScheme()` names only the
  derivation of the action nullifier. A set SHOULD name its entry scheme in its own metadata; [ERC-8084](https://ercs.ethereum.org/ERCS/erc-8084)
  is a place for such metadata.
- **Scope:** a field element naming the domain in which a person may act once. Examples: one community, one proposal,
  one airdrop. A consumer usually has its own local scope identifier (for example a `uint64` scope id) and maps it to
  the field element as described under "Scope collisions" in the Security Considerations. "Scope" in this standard
  always means the field element.
- **Nullifier:** a deterministic function of the person's identity secret and the scope. It is unique per (identity,
  scope) and unlinkable across scopes without the secret; a person with two identities has two nullifiers (see "One
  identity per person per context"). A set or consumer that stores nullifiers as `bytes32` converts them to
  `uint256` by zero extension, never by reduction.
- **Message:** a field element the proof is bound to: the hash of the action, including the context the action
  belongs to. A consumer that derives the message from bytes SHOULD use `uint256(keccak256(bytes)) >> 8`,
  Semaphore's convention, so that implementations agree.
- **Context:** a `bytes32` chosen by a consumer of nullifiers (a registration, an epoch, a proposal). The `context`
  argument of `IAdmissionRule.check` is 0 at `Phase.Join` and otherwise SHOULD equal the registry context under
  which the consumer records the action.
- **Pseudonym:** the person's nullifier in a scope, used by admission rules as the person's identifier in that scope.
- **Consumer:** the contract that calls a verifier, a registry or a rule, and owns the action (a vote, a claim, a
  join).

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
- The nullifier derivation MUST be a pseudorandom function of `scope` keyed by the identity secret, so that two
  nullifiers of one identity in two scopes are unlinkable without the secret. A combination that can be inverted
  (`secret + scope`, `secret XOR scope`) is not compliant.
- It MAY revert instead of returning false, with `MalformedProof`, `ValueOutOfField` or an error of an underlying
  verifier. Consumers MUST treat a revert as "not valid", and MUST NOT retry with reduced values.
- It MUST NOT reduce out-of-field values silently.
- It MUST NOT keep state of its own, and its result MUST depend only on its arguments, on the published root
  history of the set it verifies against (a root-age bound included) and on proof facts an append-only registry has
  recorded for exactly these five arguments (how staged hash-based verifiers publish large proofs), never on the
  caller, the transaction origin or the remaining gas, so the same proof verifies the same way for everyone.
  Use-once is the caller's (`INullifierRegistry`), and so is the choice of acceptable `setRoot` values.
- An implementation defines how each public value is encoded for its proof system: as one field element in a large
  field, or as several limbs in a small one (hash-based systems). `ValueOutOfField` refers to the bound of that
  encoding. Whatever the encoding, `scope`, `message` and `nullifier` MUST keep at least 128 bits of entropy end to
  end; a system that can only bind 31 or 64 bits per element MUST use several.
- The proof is opaque bytes of any length. The function MAY call other contracts. Callers MUST NOT assume a fixed proof
  size or gas budget.
- `personhoodScheme` names the proof system and the nullifier derivation, so clients can produce matching proofs.
- `PersonhoodVerifierDeployed` SHOULD be emitted once, at deployment or, behind a proxy, at initialisation.

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
- A consumer MUST call `consume` itself, from the contract that owns the action. Calling it through a permissionless
  intermediary (a public multicall, a generic router or forwarder, a relayer contract) hands the namespace to every
  user of that intermediary, who can then consume any nullifier without a proof.
- Entries MUST NOT be removable. A registry SHOULD be immutable code (no proxy, no upgrade path), since an upgrade
  could remove entries. A consumer SHOULD hold its registry address as an immutable; a consumer that can change it
  MUST NOT let the change reopen a context it has already used, because a fresh registry is empty and a swap is
  equivalent to deleting every entry.
- The `context` and the `scope` a consumer passes MUST come from the consumer's own state or code (an existing
  proposal, the current epoch), never from a free value in calldata; a caller who chooses the context chooses how
  often they may act. State that any caller can create (a community, a label, a round) counts as calldata here: a
  scope or context derived from it MUST include an identifier the consumer assigned itself, never only the
  caller's input.
- A consumer MUST verify the personhood proof, then consume the nullifier, then perform the action, all in one
  transaction, so that a reverted action leaves no entry and a successful action cannot be repeated. It MUST NOT
  record a nullifier whose proof did not verify: an entry point that skips invalid entries instead of reverting (a
  batch for relayers) verifies before it consumes.
- A consumer that lets a person change an earlier action (a revote) MUST NOT consume per change. It consumes once,
  or keeps its own versioning. The version (or nonce) of the change MUST be inside what `message` commits to;
  otherwise an earlier proof, which any relayer may hold, stays valid and rolls the action back.
- Contexts SHOULD be domain-separated, for example `keccak256(abi.encode("join", epoch))` rather than the bare epoch
  number, so that two kinds of action in the same consumer never share a context by accident. A consumer that hosts
  more than one scope MUST include the scope's identifier in every context.

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
- `context` is 0 at `Phase.Join`. At `Phase.Epoch` and `Phase.Action` the consumer defines it (for example a hash of
  the epoch or of the action), and documents it, either in the rule's `describe` parameters or in its own
  documentation.
- When one piece of evidence must not admit two persons in the same context (one NFT, one wallet), the rule returns a
  non-zero `ruleNullifier` derived from that evidence and `context`, and the caller MUST record it as used. Otherwise
  the rule returns 0. A non-zero `ruleNullifier` SHOULD be domain-separated by the rule's `kind` and `version`, and
  SHOULD include `scope` at `Phase.Join` (where `context` is 0), so that one piece of evidence admits one person per
  scope rather than one person across all scopes of a consumer.
- The caller MUST record rule nullifiers under a context that is domain-separated from the contexts of person
  nullifiers, and SHOULD separate it per rule (for example `keccak256(abi.encode("rule", rule, context))`). A rule
  receives the joining person's pseudonym and may return any value; recorded in the person context, a rule's return
  value can mark a chosen person's pseudonym as consumed and exclude that person.
- Evidence that can be observed before it takes effect (anything in calldata) MUST be bound to `scope`, `pseudonym`,
  `phase` and `context`, and a rule MUST refuse evidence that is not. Unbound evidence can be copied from the
  transaction pool and submitted with another person's personhood proof: that person is admitted on it and its rule
  nullifier is consumed, so the evidence's rightful holder is shut out.
- A caller that promises anonymous actions MUST use only zero-knowledge evidence at `Phase.Action`. Public evidence at
  that phase links the action to the evidence. Public evidence at `Phase.Join` or `Phase.Epoch` links the pseudonym
  itself to the evidence, and with it every action the pseudonym ever takes in the scope; a caller that promises an
  anonymous scope MUST use zero-knowledge evidence at every phase.
- A consumer that admits persons at `Phase.Join` and lets them act under other scopes MUST link each action to the
  admission by a proof, never by a pseudonym named in calldata: either the action proof is verified against a set
  of the admitted members (a second personhood set whose root the consumer maintains), or join and actions share
  one scope, so that the acting nullifier is the admitted pseudonym, and the consumer documents that actions in the
  scope are linkable. `isConsumed` in another consumer's namespace states that a pseudonym acted there, not that
  the present caller is that pseudonym.
- `describe` returns the rule kind, the version of its evidence and parameter format, and its parameters. Clients
  SHOULD show them to a person before the person joins. A rule's parameters SHOULD be immutable; a rule whose
  parameters can change MUST emit `AdmissionRuleConfigured` on every change and MUST increase `version` when the
  meaning of `evidence` or `params` changes.
- A rule MUST NOT return or imply a weight.

### Interface identifiers

| Interface | ID |
|---|---|
| `IPersonhoodVerifier` | `0x461ae3b6` |
| `INullifierRegistry` | `0xcafeb4e0` |
| `IAdmissionRule` | `0x4d42cf5d` |

Each identifier is the XOR of the interface's own function selectors (`Phase` is encoded as `uint8`):
`0x12724690 ^ 0x5468a526`, `0x98017489 ^ 0x52ffc069` and `0x57934ba2 ^ 0x1ad184ff`.

Implementations MUST return true from `supportsInterface` for their interface's identifier and for
[ERC-165](https://ercs.ethereum.org/ERCS/erc-165)'s. `supportsInterface` is a discovery aid, not a security control: a consumer MUST NOT infer
from it that a contract honours this specification.

## Rationale

### Design choices

- **Field elements, opaque proofs.** The scope, message, nullifier and root are field elements, because every SNARK
  nullifier scheme needs them; the proof stays bytes. A hash-based (post-quantum) verifier can implement the same
  interface. Its proofs are tens to hundreds of kilobytes and need several contracts, so no size or gas bound is
  fixed.
- **Stateless verifier, separate registry.** Keeping use-once out of the verifier lets one verifier serve every scope.
  A consumer can also choose its contexts (per epoch, per proposal) without a new verifier.
- **Namespacing by `msg.sender`.** As in [ERC-7812](https://ercs.ethereum.org/ERCS/erc-7812)'s key isolation, no consumer can mark another's
  nullifiers. Without it, a shared registry would let anyone block a nullifier in someone else's context.
- **Rules return a nullifier instead of keeping state.** A rule deployment is shared by many scopes; the caller
  decides where the rule nullifier is recorded.
- **Phases.** When a condition is checked decides what it leaks. A check per action is only private as a
  zero-knowledge proof. The phase is a parameter of the scope, not of the rule.
- **Two sets, one join per context.** A consumer that admits persons per epoch keeps the admitted members as a second
  personhood set: one set per consumer and epoch, entered once by one proof against the first set, under a join
  nullifier that derives from the person's secret and the context. Every action is a membership proof in the second
  set and names neither the person's registration, nor the join nullifier, nor the joining transaction; the only
  public link is "a member of this epoch". The exception is a rule with public evidence, which links one person's
  memberships across epochs (see the admission rule above).
- **No weight.** "One person, one vote" is the purpose of the standard. Token-weighted voting has its own standards
  ([ERC-1202](https://ercs.ethereum.org/ERCS/erc-1202), governor contracts).

### Related work

- **ERC-7812 (ZK Identity Registry):** a singleton registry stores abstract statements as key/value pairs in a sparse
  Merkle tree; keys are isolated per registrar (`hash(msg.sender, key)`), and registrars define their own statements
  and circuits. It is complementary: a personhood set's root, a scope's snapshot root or a certificate root can be
  stored as an ERC-7812 statement by a registrar, so other applications can prove facts about it against one global
  root. This standard covers what ERC-7812 leaves to registrars: the personhood statement, scoped nullifiers and
  admission rules. It follows ERC-7812's isolation rule (`INullifierRegistry` namespaces) and its advice to pass a
  root as a parameter and check it against the source's history instead of reading the current root on chain.
- **ERC-1202 (Voting Standard):** its votes carry a weight, its events index the voter address, and it assumes one
  address casts one decision. Each contradicts the goal here: no weight, no address, one person instead of one
  address. A personhood-based voting contract could still expose ERC-1202-style views with weight 1, never the voter.
- **[ERC-7524](https://ercs.ethereum.org/ERCS/erc-7524) (PLUME):** a deterministic signature on secp256k1 whose output is a nullifier for an
  Ethereum account, provable in a SNARK without revealing the account. It fits `IAdmissionRule` as a holder rule
  ("this person controls an address in the holder snapshot"), with the PLUME nullifier for (address, context) as the
  `ruleNullifier`, without linking the wallet to the person. It does not replace personhood: one person can hold many
  addresses. PLUME notes that a quantum adversary who breaks ECDSA could recompute nullifiers; the same applies to
  such a rule.
- **Generic proof verification.** [ERC-1922](https://ercs.ethereum.org/ERCS/erc-1922) and [ERC-1923](https://ercs.ethereum.org/ERCS/erc-1923) (zk-SNARK verifier and
  verifier registry), [ERC-7913](https://ercs.ethereum.org/ERCS/erc-7913) (signature verifiers for keys without an address), ERC-8084 (proof
  metadata), the proposal "ZK Proof Verification" (pull request 1238 of the ERCs repository, a proof-system-agnostic
  `verifyProof(bytes publicInputs, bytes proof)` in the [ERC-1271](https://ercs.ethereum.org/ERCS/erc-1271) pattern) and
  [ERC-8262](https://ercs.ethereum.org/ERCS/erc-8262) (a compliance oracle with a typed proof router, including membership and non-membership
  proofs): the verifier behind `IPersonhoodVerifier` may follow any of them. This standard fixes only the personhood
  statement, its public values and its nullifier; a generic verifier can sit behind it.
- **Nullifiers for agents.** The proposal "Unclonable Agent Execution Credentials" (pull request 1953 of the ERCs
  repository) uses single-use zero-knowledge nullifiers for delegated agent runs, with a chain-independent nullifier
  and a chain-bound capability. It arrives at the same split between the nullifier and its binding, for agents
  instead of persons.
- **Research.** "Proof-of-Uniqueness: Sybil-Resistant Privacy-Preserving Decentralized Identity through
  Threshold-OPRF and zk-SNARK Registry" (IACR ePrint 2026/1725) describes a registry with one record per person keyed
  by a nullifier, with an issuer-signed credential and a threshold OPRF as the root of uniqueness. Its registry is a
  personhood set in the sense of this standard, with a different entry scheme; the interfaces here would let a
  consumer use such a set and a passport-based set alike.
- **Other standards that touch this one:** [ERC-5732](https://ercs.ethereum.org/ERCS/erc-5732) (commit interface): commit-reveal consumers may
  reuse its commitment convention; its event, indexed by address, does not fit anonymous consumers.
  [ERC-8143](https://ercs.ethereum.org/ERCS/erc-8143) and [ERC-7734](https://ercs.ethereum.org/ERCS/erc-7734): personhood or identity as a record per address, the
  opposite binding to this standard's. [ERC-5564](https://ercs.ethereum.org/ERCS/erc-5564) and [ERC-6538](https://ercs.ethereum.org/ERCS/erc-6538) (stealth addresses):
  for consumers that pay out or issue something to an address. [ERC-7574](https://ercs.ethereum.org/ERCS/erc-7574) and the soulbound token
  standards ([ERC-5192](https://ercs.ethereum.org/ERCS/erc-5192), [ERC-5484](https://ercs.ethereum.org/ERCS/erc-5484), [ERC-4973](https://ercs.ethereum.org/ERCS/erc-4973)): out of scope;
  consumers may use them for non-transferable badges. [ERC-4824](https://ercs.ethereum.org/ERCS/erc-4824): metadata of a consuming community.
- **Semaphore (v4):** groups with an admin, `addMember`, and `validateProof(groupId, proof)`, where the proof carries
  `merkleTreeRoot`, `nullifier`, `message` and `scope`. `IPersonhoodVerifier` generalises its verification: a set
  root instead of a group id, the scheme named, the proof opaque. A Semaphore group whose members are one per person
  is a personhood set, and an adapter over Semaphore's `verifyProof` (a view) implements `IPersonhoodVerifier`.
  Semaphore's own nullifier check (`validateProof` stores it) consumes per (group, scope): its scope is this
  standard's context, and one scope cannot have several contexts there.

## Backwards Compatibility

New interfaces only. An existing verifier and existing admission conditions can be exposed through adapters without
redeployment. A consumer's internal nullifier records cannot be turned into a shared `INullifierRegistry` that way;
a registry has to be deployed.

## Test Cases

The reference implementation ships Foundry tests (`assets/erc-####/test/`) that check: the three ERC-165 identifiers
equal the XOR of the selectors and the values listed above; the minimal implementations answer `supportsInterface`
for their own identifier and ERC-165's and refuse others; the registry namespaces by consumer (two consumers can
consume the same value, one consumer cannot consume it twice in one context, other contexts are independent); and,
as a fuzz test over consumer, context and nullifier, every second consumption reverts with
`NullifierAlreadyConsumed` carrying the exact triple. A second file exercises the consumer pattern against a
hash-binding stub verifier (message binding with the beneficiary, verify-then-consume in one transaction, a verifier
revert leaving no entry). A third and a fourth file pair, for the rules that address consumers and rules, a consumer
that breaks a rule with one that follows it, and show the attack succeeding against the first and failing against the
second: a permissionless forwarder as the registry's caller, a rule nullifier recorded in the person context, evidence
not bound to the pseudonym, a scope without the chain id on two chains, a revote message without the version, a
caller-chosen context, a registry swap, two sets or a set swap in one context, an identity replaced inside a context,
a join pseudonym named in calldata, a message without the context, consume before verify, and a tenant-chosen scope.
A fifth file is an invariant suite for the registry.

## Reference Implementation

`assets/erc-####/src/` contains the three interfaces, a `MinimalNullifierRegistry` (the namespaced registry in a
nested mapping, no owner, no removal), an `AllowAllRule` and a `RejectAllPersonhoodVerifier` that fix the ERC-165
identifiers. They compile with Solidity 0.8.24 and above and depend on nothing but the interfaces themselves.

Henosis, a governance layer on Shibarium, plays the three roles in its own contracts with its own ABIs, which is how
the interfaces were found: its anonymous voting verifies membership in a per-epoch group filled from its identity
registry (the role of `IPersonhoodVerifier`), and records registrations, joins and proposals once per nullifier (the
role of `INullifierRegistry`); votes are versioned to allow revoting. Admission rules (`IAdmissionRule`) are planned
behind a governed slot there. Adapters exposing these interfaces are part of its first release.

## Security Considerations

- **The set decides Sybil resistance.** This standard proves membership in a set, not that the set has one entry per
  person. A consumer accepts only `setRoot` values from a source it trusts, and checks them against that source's
  root history. A contract that publishes a personhood set SHOULD expose `isKnownSetRoot(uint256 root) returns
  (bool)`, so consumers can check a root without reading the source's internals; this optional view is not a fourth
  interface and does not claim compatibility with ERC-7812, whose root query answers for its own statement tree.
- **Root freshness.** A root that was valid once stays provable forever. A consumer that accepts old roots accepts
  members removed since. Consumers SHOULD bound the age of acceptable roots, and set publishers SHOULD record the
  time each root became known. A consumer that verifies personhood only once, at joining, has no way to drop a person
  removed from the set afterwards (a revoked credential, a duplicate found later); a consumer whose scope lives
  longer than its set's revocation latency SHOULD re-verify at least once per epoch.
- **One identity per person per context.** A nullifier is unique per (identity, scope), not per (person, scope). A
  person with two identities has two nullifiers in every scope: two sets accepted side by side, a set replaced while
  a context is open, a verifier with another nullifier scheme installed mid-context, or an identity replaced inside
  one set (recovery, key rotation, re-enrolment). Within one context a consumer MUST accept proofs against exactly
  one personhood set and one nullifier scheme, and MUST apply a change of either only when a new context opens. A
  set in which a person can replace their identity commitment MUST document it. A consumer of such a set MUST
  either bind each context to one root, fixed when the context opened, or key its use-once check on a per-person
  value of the set's entry scheme that survives the replacement, never on the action nullifier alone.
- **Scope collisions across chains and contracts.** A nullifier is unique per scope. Two applications that use the
  same scope value link a person's actions across them, and can block each other. Scopes MUST include the chain id
  and the consuming contract's address, for example `scope = hash(chainId, consumer, localScope)` reduced into the
  field. The chain id is not optional: deterministic deployment gives a consumer the same address on several chains,
  and with the same scope the same proof is valid on each of them, so one proof claims once per chain. Where an
  external system fixes the scope so that it cannot carry the chain id, the consumer MUST put the chain id into
  `message`; this stops the replay but not the linkage, since the person then has the same pseudonym on every chain.
- **Pseudonym linkability within a scope.** Within one scope a person always presents the same nullifier, so every
  action in that scope is linkable to every other by anyone who reads the chain. That is what makes the nullifier a
  pseudonym. A consumer that needs actions to be unlinkable from each other (one proposal from the next) MUST use a
  distinct scope per action, and record consumption under the matching context.
- **Message binding.** Without a message bound in the proof, a proof can be front-run and reused for another action.
  Consumers MUST compute `message` from the action themselves, never accept it from the caller. Everything the
  action's effect depends on, a beneficiary address included, MUST be inside what `message` commits to, and so MUST
  the context the nullifier is consumed under, unless the scope is distinct per context; otherwise a copied
  transaction can redirect the effect, or carry the proof into another context, while the proof stays valid. A
  proof is a bearer instrument: once it has left the prover, anyone may submit it at any time, so whatever must not
  be replayable later (a version, a deadline, an epoch) also belongs in `message`.
- **Uniqueness rests on the nullifier, never on the proof bytes.** Proofs in many systems are malleable: a different
  byte string proves the same statement. A consumer MUST NOT use a hash of the proof as a replay guard; the nullifier
  recorded in the registry is the only replay guard this standard provides.
- **Empty leaves of a set.** A personhood set committed to by a Merkle root has padding leaves. If the padding value
  is a reachable identity commitment (zero, or the hash of a known preimage), whoever knows its preimage is a member
  of every set that is not full. A set MUST use a padding value with no known preimage as an identity, and SHOULD
  document it with its entry scheme.
- **Linkage through admission rules.** Public evidence at `Phase.Action` links actions to evidence. The set of
  attestations or rule nullifiers at a pseudonym can be a fingerprint. Clients SHOULD show this before joining.
- **Field overflow.** A value at or above the field modulus, accepted and reduced, gives a second encoding of the same
  nullifier. Verifiers MUST refuse it (`ValueOutOfField`), and registries SHOULD as well.
- **Verifier replacement.** A consumer that can swap its `IPersonhoodVerifier` can swap its security. Replacement
  SHOULD go through a time lock with an objection window, and MUST apply only from a context boundary. A verifier is
  called as an external contract and MAY revert or consume arbitrary gas; a consumer MUST NOT let a verifier's
  failure strand funds or lock a state machine it cannot leave by another path.
- **Mutable rules.** A rule whose parameters change after persons joined changes the terms they joined under.
  Rules SHOULD be immutable deployments; a scope that admits a mutable rule SHOULD treat a parameter change like a
  rule rotation.
- **Deterministic nullifiers after a key compromise.** Whoever learns an identity secret can compute all of that
  person's nullifiers, past and future, and link them. The same holds for PLUME-style nullifiers under a quantum
  adversary.
- **No sender binding.** Consumers SHOULD keep their action entry points free of `msg.sender` binding. Then relayers,
  account abstraction and fresh addresses all work, and a person's actions are not linked through a submitting
  account. This includes recovery and refund paths: a deposit refund SHOULD be claimable by a proof of the new
  secret, with the payout address chosen at claim time, not recorded at request time.
- **Weight.** Treating a rule's evidence (a balance) as weight breaks the purpose of this standard. Rules MUST NOT
  expose a weight, and consumers MUST NOT derive one from them.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE-CC0).
