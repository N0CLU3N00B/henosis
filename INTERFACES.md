# Interfaces for scoped proof of personhood

Henosis fixes three small, neutral interfaces between personhood, nullifiers and the rules of a community. They do
not depend on the proof system, on the kind of credential that put a person into the set (an identity document, a
presence event, vouching) or on the chain. The full proposal is the draft ERC
[`erc/scoped-proof-of-personhood.md`](erc/scoped-proof-of-personhood.md). This page describes the three interfaces in
words; the signatures are listed for reference. All three support ERC-165.

## Terms

- **Personhood set:** a set of identity commitments, at most one per person, committed to by a root (`setRoot`).
- **Scope:** a field element naming the domain in which a person may act once: a community, a proposal, an
  airdrop.
- **Nullifier:** a deterministic function of the person's identity secret and the scope. It is unique per person and
  scope, and without the secret it cannot be linked across scopes.
- **Message:** the value a proof is bound to, for example an action hash or a vote option.
- **Context:** a value chosen by whoever consumes nullifiers: a registration, an epoch, a proposal.
- **Pseudonym:** a person's nullifier in a scope, used as their identifier there.

## 1. IPersonhoodVerifier

**Purpose:** checks a proof that the prover is one member of a personhood set, and gives the prover's nullifier for
one scope, bound to one message.

**What a valid proof states:**
1. the prover knows the secret of an identity in the set with root `setRoot`;
2. `nullifier` is that identity's nullifier for `scope`;
3. the proof is bound to `message`, so it cannot be reused for another action.

**Behaviour:**
- The verifier keeps no state; the same proof always verifies the same way.
- Making sure a nullifier is used only once is the caller's job (see `INullifierRegistry`). So is the choice of
  which `setRoot` values are acceptable.
- Values outside the proof system's field are refused, never silently reduced.
- The proof is opaque bytes of any length, so a post-quantum, hash-based verifier can implement the same interface.
- `personhoodScheme` names the proof system and the nullifier derivation, so clients can produce matching proofs.

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

## 2. INullifierRegistry

**Purpose:** records which nullifiers have been used, so each person acts at most once per context.

**Behaviour:**
- Every entry is keyed by the consumer (the calling contract), the context and the nullifier. A consumer writes only
  in its own namespace and can never mark or unmark another consumer's nullifiers.
- Consuming a nullifier twice in the same context fails.
- Entries are never removed.
- A consumer that lets a person change an earlier action, such as a revote, does not consume per change. It consumes
  once, or keeps its own versioning.

```solidity
interface INullifierRegistry is IERC165 {
    error NullifierAlreadyConsumed(address consumer, bytes32 context, uint256 nullifier);
    event NullifierConsumed(address indexed consumer, bytes32 indexed context, uint256 indexed nullifier);
    function consume(bytes32 context, uint256 nullifier) external;
    function isConsumed(address consumer, bytes32 context, uint256 nullifier) external view returns (bool consumed);
}
```

## 3. IAdmissionRule

**Purpose:** one condition, in addition to personhood, that a scope places on joining or acting. Examples: holding
an asset, an attestation, vouches, a presence record, or a combination. The scope chooses its rules.

**Behaviour:**
- A rule admits or refuses. It never returns or implies a weight.
- It keeps no state, so one deployment can serve many scopes.
- **When it is checked** is chosen by the scope: once at joining, at every epoch, or with every action.
- **Evidence that must admit only one person:** when one piece of evidence (one NFT, one wallet) must not admit two
  persons in the same context, the rule returns a rule nullifier derived from that evidence. The caller records it.
- **Checks with every action:** these must use zero-knowledge evidence if actions are meant to stay anonymous.
  Public evidence at that point would link the action to it.
- `describe` returns the rule kind, the version of its format and its parameters, so a client can show a person the
  rule before they join.

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

## ERC-165 identifiers

| Interface | ID |
|---|---|
| `IPersonhoodVerifier` | `0x461ae3b6` |
| `INullifierRegistry` | `0xcafeb4e0` |
| `IAdmissionRule` | `0x4d42cf5d` |
