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
- The decryption threshold in use for a round, and the period over which
  key shares are retained, are published, since both bound the
  amount-privacy claims made elsewhere in the protocol.
- The organisations holding shares of the election authority key have
  stated, before the round opens, that they will tally it.
- The conditions under which a round's result may be described as
  representative of coinholder sentiment are stated, rather than left
  to the reader of a result.


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

### Signature Verification

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

## Election Authority Key

$\mathsf{ea}\_\mathsf{pk}$ is a per-round value: a new election authority
key is generated for each round, and it appears in the round
configuration. This document specifies the round-facing properties of
that key. The cryptographic construction — El Gamal on Pallas, the
Shamir sharing, the DLEQ proofs used at tally, and the ceremony message
flow — is specified separately in [^ea-ceremony].

### Threshold

The decryption threshold $t$ is the number of key-share holders that
must cooperate to decrypt. It determines the strength of every
amount-privacy claim in [^voting-protocol], since any $t$ holders can
decrypt an individual share as readily as the aggregate.

A deployment MUST publish the value of $t$ and the number of key-share
holders $n$ in use for a round, and these MUST match the values the
ceremony actually used. Specifications and deployments have differed on
this value; publishing it per round makes a divergence visible rather
than latent.

### Share Generation

A ceremony that generates the key at a single party and distributes
shares from it — a trusted dealer — MUST be treated as giving that
party the full election authority secret key for the duration of the
ceremony. The claim that no single party holds
$\mathsf{ea}\_\mathsf{sk}$ holds only after the ceremony completes, and
only if the dealer destroyed its copy, which no other party can verify.

A deployment using a trusted dealer MUST disclose this in the round
configuration or accompanying documentation, MUST identify the party
acting as dealer, and SHOULD adopt distributed key generation or
publish verifiable secret sharing commitments, so that share holders
can confirm their shares are consistent with $\mathsf{ea}\_\mathsf{pk}$
without trusting the dealer.

### Key Retention

Key shares MUST NOT be retained indefinitely.

Each share holder MUST destroy its share of $\mathsf{ea}\_\mathsf{sk}$
once the round is finalised and the tally published. A deployment MUST
publish the retention period it applies and the point at which
destruction occurs.

Retaining shares for possible future retally or audit is not a
sufficient reason to keep them. The encrypted shares of every
individual vote remain on the vote chain permanently, and the
encryption is not post-quantum. A retained key share is therefore not a
dormant convenience: it is a live capability, held indefinitely,
against a permanent public record of individual voters' balances. The
audit properties retention is intended to preserve are available
without it, because the partial decryptions and their DLEQ proofs are
published on chain and can be re-verified at any time without
re-deriving the key.

Where a deployment concludes that retention is nonetheless required, it
MUST state for how long, and MUST treat that period as the period over
which its amount-privacy claims hold — not the duration of the round.

### Ratification

Administrator attestation ([Configuration Authentication]) establishes
that a round's parameters are correct. It does not establish that the
round will be tallied. Those are different parties: administrators
configure a round, key-share holders decrypt its result. A round can be
correctly configured, voted in, and never opened.

Each holder of a share of $\mathsf{ea}\_\mathsf{sk}$ for a round MAY
publish a **ratification**: a signed statement that it holds a share
for that round and will participate in the tally.

The bytes covered by a ratification signature are the concatenation, in
this order, of:

| Component | Width |
|---|---|
| The ASCII string `ZcashVotingRoundRatification:v1` | 31 bytes |
| `vote_round_id` | 32 bytes |
| `ea_pk` | 32 bytes |

Binding to $\mathsf{ea}\_\mathsf{pk}$ as well as to the round
identifier is necessary: a ratification that covered only the round
would carry over to a round rekeyed after the fact, which is the case
it most needs to exclude.

A round MUST NOT open for voting unless at least $t$ distinct key-share
holders have published valid ratifications for it, where $t$ is the
decryption threshold for that round ([Threshold]). A deployment MUST
publish the ratifications it collected, and SHOULD obtain them from all
$n$ holders rather than the minimum.

The threshold for ratification is $t$ rather than a separate parameter
because below $t$ the question does not arise: a round ratified by
fewer than $t$ holders cannot be tallied even if every ratifying holder
honours its statement. Requiring $t$ makes the published ratifications
a statement that the round is tallyable, not merely that some holders
are willing.

A ratification is a statement of intent, not an enforceable
commitment. A holder can ratify and then decline to participate, and
nothing in this document prevents that. What ratification provides is
that the decision is made and published *before* voters commit their
balances, rather than discovered afterwards — and that a holder
declining to tally a round it ratified is visibly departing from a
signed statement rather than exercising an unstated discretion. See
[Why Ratification Precedes Voting].

## Representativeness

This section constrains how a round's result may be described. It is
normative because the description is the product: a poll exists to be
cited.

A round's result MUST NOT be described as representative of coinholder
sentiment unless, for the full duration of the round, at least $k$
independent conforming wallet implementations were available to voters,
where $k$ is a deployment parameter that MUST be at least 2.

Two implementations are **independent** for this purpose if neither
derives its share decomposition ([^voting-protocol]), its server
selection, or its submission scheduling from the same library as the
other. Wallets that share a voting library are one implementation for
this test regardless of how they are branded, because they share the
behaviour that determines what a voter discloses. [^wallet-api]
requires a wallet to disclose whether it implements these behaviours
itself or consumes them, which makes the test checkable from published
information rather than from inspection.

A deployment MUST publish, for each round, the implementations it
counted and the basis on which it considered them independent.

Where the condition is not met, a round remains valid and its result
remains verifiable; what is not available is the claim that the result
measures sentiment rather than the behaviour of the voters who had
access to the one client that existed. See
[Why Implementation Diversity Is a Precondition].

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

## Why Ratification Precedes Voting

A voter deciding whether to participate is deciding whether to expose a
quantity — their balance at the snapshot, to the extent the protocol
permits — in exchange for influence over an outcome. That trade is only
available if the outcome will be produced.

Without ratification the voter has no way to check the second half. The
round configuration names $\mathsf{ea}\_\mathsf{pk}$, but a public key
is not a statement by anybody that they will use the corresponding
shares. A voter can verify that a round is correctly configured and
still be voting into a round that no key-share holder intends to open.

Placing ratification before the round opens rather than at tally time
is the whole of the requirement. A statement collected afterwards
records what happened; a statement collected beforehand is an input the
voter can act on.

## Why Implementation Diversity Is a Precondition

A single implementation is a single point of behavioural failure that
no amount of protocol correctness compensates for.

Where one client is the only way to vote, its defaults are the
protocol as experienced by every voter. If it batches share
submissions, every voter batches. If it delegates the full balance
rather than a subset, every voter does. If it reimplements server
selection and so does not inherit a library's constraints, that gap
applies to the entire electorate at once. Each of these has occurred in
deployed voting clients, and in each case the protocol documents
permitted the correct behaviour while the sole available client did
something else.

Diversity does not prevent any of that. What it does is make the
failure partial and detectable: two independent implementations that
disagree about what a conforming client does expose the ambiguity, and
voters retain a choice that does not depend on one vendor's judgement.

The requirement is placed here, on the round, rather than in
[^wallet-api], because it is not a property any single wallet can
satisfy. A wallet cannot make itself diverse. It is a property of the
round's circumstances, and therefore of whether the round's result
supports the claim that is made about it.

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

A deployment MUST publish, for each round, in addition to the
parameters required elsewhere in this document:

| Parameter | Constraint |
|---|---|
| $m$ | Administrator signature threshold; MUST be at least 2. See [Signature Verification]. |
| $t$, $n$ | Decryption threshold and key-share holder count. See [Threshold]. |
| $k$ | Independent conforming wallet implementations required; MUST be at least 2. See [Representativeness]. |
| Ratifications | The key-share holder ratifications collected for the round. See [Ratification]. |


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
- The election authority key ceremony is specified separately in
  [^ea-ceremony], which is not currently an open proposal. The
  round-facing requirements in [Election Authority Key] depend on it, so
  it needs to be revived and completed. In particular the trusted dealer
  construction, and the recommendation there that shares be retained
  indefinitely, should be revisited.
- $\mathsf{NF}(H)$ is defined over Orchard nullifiers only, matching the
  current protocol's restriction to the Orchard pool. Extension to
  other pools would require a corresponding extension here.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-0200]: [ZIP 200: Network Upgrade Mechanism](zip-0200)

[^protocol]: [Zcash Protocol Specification](protocol/protocol.pdf)

[^rfc8032]: [RFC 8032: Edwards-Curve Digital Signature Algorithm (EdDSA)](https://www.rfc-editor.org/rfc/rfc8032)

[^wallet-api]: [Shielded Voting Wallet API](draft-valargroup-shielded-voting-wallet-api)

[^voting-protocol]: [Draft ZIP: Shielded Voting Protocol](draft-valargroup-shielded-voting)

[^voting-setup]: [Draft ZIP: Zcash Shielded Coinholder Voting](draft-valargroup-shielded-voting-setup)

[^pir-governance]: [Draft ZIP: Private Information Retrieval for Nullifier Exclusion Proofs](draft-valargroup-nullifier-pir)

[^ea-ceremony]: [Draft ZIP: Election Authority Key Ceremony](draft-valargroup-ea-key-ceremony)
