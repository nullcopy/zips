    ZIP: Unassigned
    Title: Shielded Voting: Poll Configuration and Snapshot
    Owners: John Boyd <john@coldnoise.net>
    Status: Draft
    Category: Standards
    Created: 2026-09-16
    License: MIT
    Pull-Request: <https://github.com/zcash/zips/pull/????>


# Terminology

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", "MAY" and
"RECOMMENDED" in this document are to be interpreted as described in
BCP 14 [^BCP14] when, and only when, they appear in all capitals.

The term "network upgrade" in this document is to be interpreted as
described in ZIP 200 [^zip-0200].

**Voting round**: a single instance of coinholder polling, anchored to a
Zcash mainnet snapshot and conducted on its own vote chain.

**Snapshot**: the state of the Zcash Orchard shielded pool at a chosen
mainnet block height, against which voting eligibility and weight are
determined.

**Snapshot roots**: the two commitment values that summarise the
snapshot — the Orchard note commitment tree root
($\mathsf{nc}\_\mathsf{root}$) and the nullifier non-membership Indexed
Merkle Tree root ($\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$).

**Poll runner**: the party that selects the snapshot height and prepares
a round's configuration.

**Administrator**: a party whose signature over a round configuration is
recognised by wallets.

**Verifier**: any party that independently checks a round's
configuration. A verifier need not be an administrator, a validator, or
a voter.


# Abstract

This ZIP specifies what a shielded voting round is: how its Zcash
mainnet snapshot is chosen, how the two snapshot roots that anchor every
subsequent proof are derived, and how any party can independently
recompute them.

The snapshot roots are deterministic functions of Zcash mainnet
consensus state. This ZIP requires that they be treated as such:
computed independently by administrators before attestation, published
with the parameters needed to reproduce them, and recomputable by any
verifier from a Zcash full node.


# Motivation

Every proof in the shielded voting protocol [^voting-protocol] is
anchored to two values: $\mathsf{nc}\_\mathsf{root}$, which determines
whose notes may vote, and
$\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$, which determines
which notes were unspent at the snapshot and therefore prevents a note
from voting twice.

Neither value is currently produced or checked by the voting chain. Both
are supplied as input by the party that proposes a round. The
coinholder-voting setup draft [^voting-setup] states the relevant
property plainly — "These values are deterministic given the height;
they are derived from Zcash mainnet state" — and then requires no party
to perform that derivation. Its verification procedure specifies
per-transaction proof checking, nullifier set integrity, and tally
correctness, and concludes that "no full-node operator needs to trust
any other participant". That conclusion does not follow: every input it
enumerates is anchored to two values that are not on the vote chain, are
supplied by a single party, and are recomputed by no one.

The consequence for
$\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ is a double-voting
path. If the nullifier set underlying the root omits a nullifier that
was present on Zcash mainnet at the snapshot height, the holder of the
corresponding spent note can obtain a non-membership proof for it and
cast a second vote with a note they had already spent. An omission of
this kind need not be deliberate: it can arise from an ingest pipeline
that does not correctly handle a Zcash chain reorganisation, a case no
current document addresses.

This ZIP does not propose that the vote chain validate the roots,
though that is the complete fix and is discussed in [Rationale]. It
specifies the roots precisely enough that independent verification is
possible today, and that consensus validation can be added later
without redesign.


# Requirements

- The snapshot roots are deterministic functions of Zcash mainnet
  consensus state and a snapshot height, and are reproducible by any
  party with a Zcash full node.
- A round's configuration carries every parameter needed to reproduce
  the snapshot roots.
- Administrators recompute the snapshot roots independently before
  attesting to a round.
- A verifier who is not an administrator, validator, or voter can
  confirm that a round's snapshot roots are correct.
- A snapshot is anchored to a specific Zcash block, not only to a
  height, so that a chain reorganisation affecting the snapshot is
  detectable.


# Non-requirements

- Validation of the snapshot roots by vote chain consensus. This is the
  structural remedy and is discussed in [Rationale], but it requires
  validators to follow Zcash mainnet state and is out of scope here.
- The vote chain's consensus mechanism and validator set, specified in
  [^voting-setup].
- The voting protocol's circuits and transaction types, specified in
  [^voting-protocol].
- The private information retrieval scheme by which wallets obtain
  non-membership proofs, specified in [^pir-governance]. This ZIP
  defines the set; that ZIP defines private access to it.


# Specification

## Snapshot Height Selection

A voting round MUST be anchored to a single Zcash mainnet block,
identified by both its height $H$ and its block hash
$\mathsf{snapshot}\_\mathsf{blockhash}$.

The poll runner selects $H$ subject to the following constraints:

1. $H$ MUST be at or after NU5 activation, since the protocol requires
   Orchard.
2. At the time the round configuration is published, $H$ MUST be at
   least $\mathsf{min}\_\mathsf{confirmations}$ blocks below the tip of
   the Zcash mainnet best chain, where
   $\mathsf{min}\_\mathsf{confirmations}$ is a deployment parameter with
   a RECOMMENDED value of 100 (see [Deployment]).
3. $\mathsf{snapshot}\_\mathsf{blockhash}$ MUST be the hash of the block
   at height $H$ on the Zcash mainnet best chain.

Constraint 2 exists so that the snapshot is taken from a portion of the
chain that is not realistically subject to reorganisation. Constraint 3
binds the round to a specific block rather than to a height, so that a
reorganisation affecting the snapshot can be detected rather than
silently changing the meaning of the round.


## Snapshot Roots

The snapshot roots are defined as functions of Zcash mainnet consensus
state at the block identified by $(H, \mathsf{snapshot}\_\mathsf{blockhash})$.

**Note commitment tree root.**
$\mathsf{nc}\_\mathsf{root}$ MUST be the root of the Orchard note
commitment tree as of the end of block $H$, as defined by the Zcash
Protocol Specification [^protocol]. This is the same value a Zcash full
node exposes as the Orchard anchor at that height.

**Nullifier non-membership tree root.**
$\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ MUST be the root of
the Indexed Merkle Tree constructed over the set

$$\mathsf{NF}(H) = \{\, \mathsf{nf} : \mathsf{nf} \text{ is an Orchard nullifier revealed in a block at height } \leq H \text{ on the best chain} \,\}$$

using the tree construction specified in [^pir-governance].

$\mathsf{NF}(H)$ MUST contain every Orchard nullifier revealed at or
before height $H$, and MUST NOT contain any other value. In particular
it MUST NOT omit nullifiers revealed in blocks that were part of the
best chain at height $\leq H$ but were not yet processed by the
producing implementation at the time of computation.

Both roots are deterministic given $(H, \mathsf{snapshot}\_\mathsf{blockhash})$.
No party has discretion over their values.


## Snapshot Recomputation

This section specifies the procedure by which a party independently
derives the snapshot roots. It is the normative reference for what it
means for a round's roots to be *correct*.

A verifier with access to a Zcash full node MUST be able to perform the
following:

1. Confirm that the block at height $H$ on the node's best chain has
   hash $\mathsf{snapshot}\_\mathsf{blockhash}$. If it does not, the
   verification fails and the round MUST be treated as invalid (see
   [Chain Reorganisation]).
2. Derive $\mathsf{nc}\_\mathsf{root}$ as the Orchard note commitment
   tree root as of the end of block $H$.
3. Enumerate $\mathsf{NF}(H)$ by scanning every block at height
   $\leq H$ on the best chain and collecting every Orchard nullifier
   revealed in each, then construct the Indexed Merkle Tree over that
   set per [^pir-governance] and derive its root.
4. Compare both derived values against those published in the round
   configuration.

**Administrator obligation.** An administrator MUST perform this
procedure, using a Zcash full node under its own control, before
attesting to a round configuration. An administrator MUST NOT attest to
a configuration whose snapshot roots it has not independently derived.

Confirming that a configuration matches the one circulated by the poll
runner is not a substitute. The published artifact is a pair of root
values; comparing them byte-for-byte against another party's copy of
the same values establishes only that the two parties received the same
input, not that the input is correct.

**Publication.** The poll runner SHOULD publish, alongside the round
configuration, the software name and version used to derive the roots
and the resulting values, so that a verifier can distinguish a
disagreement about the snapshot from a disagreement about the
derivation.


## Chain Reorganisation

The set $\mathsf{NF}(H)$ is defined over the best chain. An
implementation that ingests blocks incrementally and does not roll back
on a reorganisation can produce a set that includes nullifiers from
blocks no longer on the best chain, omits nullifiers from blocks that
replaced them, or both.

An implementation producing
$\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ MUST handle
reorganisations of the Zcash mainnet chain. Specifically:

1. It MUST track the block hash of each ingested block, not only its
   height.
2. On detecting that a previously ingested block is no longer on the
   best chain, it MUST discard all state derived from that block and
   every block after it, and re-ingest from the last common ancestor.
3. It MUST NOT produce a root for height $H$ until it has confirmed
   that the block it ingested at height $H$ is on the current best
   chain and has hash $\mathsf{snapshot}\_\mathsf{blockhash}$.

If a reorganisation affects the block at height $H$ after a round
configuration has been published, the round MUST NOT open. If it is
already open, the round MUST be abandoned: the snapshot it is anchored
to no longer exists on the best chain, and the eligibility of every
vote cast in it is undefined.

A nullifier service that serves non-membership proofs against
$\mathsf{NF}(H)$ MUST apply the same requirements to its ingest
pipeline.


## Round Configuration

A round configuration is the document from which a wallet learns that a
round exists and what it is anchored to. It is published out-of-band;
the vote chain does not serve it.

A configuration entry for a round MUST carry every field needed to
perform [Snapshot Recomputation] and to check the result:

| Field | Type | Description |
|---|---|---|
| `auth_version` | integer | Schema version of this entry. This document defines 2. |
| `vote_round_id` | hex, 64 chars | Round identifier, as derived by the vote chain. |
| `snapshot_height` | integer | Zcash mainnet height $H$. |
| `snapshot_blockhash` | base64, 32 bytes | Hash of the block at height $H$. |
| `nc_root` | base64, 32 bytes | Orchard note commitment tree root at $H$. |
| `nullifier_imt_root` | base64, 32 bytes | Nullifier non-membership tree root at $H$. |
| `proposals_hash` | base64, 32 bytes | Commitment to the round's proposals. |
| `vote_end_time` | integer | Unix timestamp after which votes are not accepted. |
| `ea_pk` | base64, 32 bytes | Election authority public key for this round. |
| `min_confirmations` | integer | Confirmation depth used when selecting $H$ (see [Snapshot Height Selection]). |
| `signatures` | array | Administrator signatures; see [Configuration Authentication]. |

An entry with `auth_version` 1, which carries only `ea_pk` and
`signatures`, MUST NOT be accepted for a round created after this
document takes effect. Wallets MAY continue to accept version 1 entries
for rounds created earlier, and if they do MUST treat the round's
snapshot as unattested.


## Configuration Authentication

Administrators attest to a round by signing it. This section specifies
what they sign and how many signatures a wallet requires.

### Covered Bytes

For `auth_version` 2, the bytes covered by each signature are the
concatenation, in this order, of:

| Component | Width |
|---|---|
| The ASCII string `ZcashVotingRoundAttestation:v2` | 30 bytes |
| `vote_round_id` | 32 bytes |
| `snapshot_height`, big-endian unsigned | 4 bytes |
| `snapshot_blockhash` | 32 bytes |
| `nc_root` | 32 bytes |
| `nullifier_imt_root` | 32 bytes |
| `proposals_hash` | 32 bytes |
| `vote_end_time`, big-endian unsigned | 8 bytes |
| `ea_pk` | 32 bytes |
| `min_confirmations`, big-endian unsigned | 4 bytes |

All components are fixed width, so the encoding is unambiguous without
length prefixes. The domain separator distinguishes these bytes from
any other signature this key may produce.

### Verification

For each entry in a round's `signatures`, a wallet:

1. MUST resolve `key_id` to an administrator key in its trusted key
   set. If no matching entry exists, the signature MUST be treated as
   invalid.
2. MUST verify that the signature's `alg` matches the `alg` declared on
   the resolved key. If they differ, the signature MUST be treated as
   invalid.
3. MUST verify the signature over the bytes defined in [Covered Bytes].
   For `"ed25519"`, per RFC 8032 [^rfc8032].
4. MUST count at most one valid signature per distinct `key_id`.

A wallet MUST accept a round entry only if the number of valid
signatures is at least $m$, where $m$ is the administrator signature
threshold. $m$ MUST be at least 2, and MUST be configured in the
wallet's trusted key set rather than read from the round entry.

A threshold read from the document it authenticates provides no
assurance, since a party able to publish a configuration could also
set its threshold to 1.

### Administrator Obligations

An administrator MUST NOT produce a signature over a round entry unless
it has performed [Snapshot Recomputation] for that round, using a Zcash
full node under its own control, and obtained values matching
`nc_root` and `nullifier_imt_root` in the entry.

An administrator MUST NOT treat agreement with another party's copy of
the configuration as satisfying this obligation.

## Verification

A round is **well-formed** if a verifier, using only a Zcash full node
and the published round configuration, can confirm:

1. The block at height $H$ on the Zcash mainnet best chain has hash
   $\mathsf{snapshot}\_\mathsf{blockhash}$.
2. $\mathsf{nc}\_\mathsf{root}$ matches the value derived per
   [Snapshot Recomputation].
3. $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ matches the value
   derived per [Snapshot Recomputation].

Well-formedness is a property of the round's configuration, and is
independent of the verification of individual votes and of the tally,
which are specified in [^voting-protocol] and [^voting-setup]. Those
procedures verify that votes are correctly formed *with respect to* the
snapshot roots; they do not establish that the roots are correct. Both
are required before a published result can be relied upon.


# Rationale

## Why Independent Recomputation Rather Than Attestation Alone

Requiring more administrators to sign a configuration does not, on its
own, improve the correctness of the snapshot roots. If no signer derives
the roots independently, a threshold of signatures attests only that
several parties received the same document from the same source.

Recomputation is what makes attestation meaningful, which is why this
document specifies it first. A signature threshold is valuable, but it
is a multiplier on an underlying check; without the check there is
nothing to multiply.

## Why Not Consensus Validation

The complete remedy is for the vote chain to compute the snapshot roots
itself, making them consensus data rather than an input. Then no party
attests to them, and no verifier needs to trust that an administrator
performed a procedure.

This is not specified here because it requires every vote chain
validator to follow Zcash mainnet state and to implement the Orchard
note commitment tree and the nullifier tree construction — a substantial
addition to the validator's responsibilities, and a change to the vote
chain's consensus rules rather than to a configuration document.

This ZIP is written so that the change remains available. The roots are
specified as deterministic functions of consensus state, with an
explicit derivation procedure and explicit reorganisation handling.
Adding consensus validation later requires validators to implement
[Snapshot Recomputation]; it does not require redefining what the roots
are.

Until that change is made, the correctness of a round's snapshot rests
on administrators performing [Snapshot Recomputation]. This document
makes that dependency explicit rather than leaving it implicit.

## Why Bind to a Block Hash

Anchoring a round to a height alone leaves the round's meaning dependent
on which chain the reader is following. Binding to
$(H, \mathsf{snapshot}\_\mathsf{blockhash})$ makes a reorganisation
affecting the snapshot a detectable condition with a specified
response, rather than a silent change in the eligible note set.


# Deployment

- $\mathsf{min}\_\mathsf{confirmations}$: RECOMMENDED value 100 blocks.
  A deployment MAY choose a larger value. The value used for a round
  MUST be published with its configuration.
- The software and version used by the poll runner to derive the
  snapshot roots SHOULD be published with each round configuration.
- Administrators SHOULD publish the software and version they used to
  perform [Snapshot Recomputation], and the values they derived.


# Open issues

- Consensus validation of the snapshot roots by the vote chain, as
  discussed in [Why Not Consensus Validation].
- The Electoral Authority key ceremony, previously drafted separately,
  is expected to be incorporated into this document, since
  $\mathsf{ea}\_\mathsf{pk}$ is a per-round configuration value.
- $\mathsf{NF}(H)$ is defined over Orchard nullifiers only, matching the
  current protocol's restriction to the Orchard pool. Extension to
  other pools would require a corresponding extension here.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-0200]: [ZIP 200: Network Upgrade Mechanism](zip-0200)

[^protocol]: [Zcash Protocol Specification](protocol/protocol.pdf)

[^rfc8032]: [RFC 8032: Edwards-Curve Digital Signature Algorithm (EdDSA)](https://www.rfc-editor.org/rfc/rfc8032)

[^voting-protocol]: [Draft ZIP: Shielded Voting Protocol](draft-valargroup-shielded-voting)

[^voting-setup]: [Draft ZIP: Zcash Shielded Coinholder Voting](draft-valargroup-shielded-voting-setup)

[^pir-governance]: [Draft ZIP: Private Information Retrieval for Nullifier Exclusion Proofs](draft-valargroup-nullifier-pir)
