    ZIP: Unassigned
    Title: Shielded Voting Protocol
    Owners: Dev Ojha <dojha@berkeley.edu>
            Roman Akhtariev <ackhtariev@gmail.com>
            Adam Tucker <adamleetucker@outlook.com>
            Greg Nagy <greg@dhamma.works>
    Credits: Daira-Emma Hopwood
             Jack Grigg
    Status: Draft
    Category: Standards
    Created: 2026-03-04
    License: MIT
    Pull-Request: <https://github.com/zcash/zips/pull/1200>


# Terminology

The key words "MUST", "REQUIRED", "MUST NOT", "SHOULD", and "MAY" in this
document are to be interpreted as described in BCP 14 [^BCP14] when, and
only when, they appear in all capitals.

The character § is used when referring to sections of the Zcash Protocol
Specification. [^protocol]

The terms below are to be interpreted as follows. These definitions are
descriptive; the rules that govern each term are in [Specification].

Ballot

: A unit of voting weight derived from a zatoshi balance. The
  conversion from zatoshi to ballots is defined in [Ballot Scaling].

Cancellation

: A transaction by which a trustee of a pending voting round, or the
  round's creator, withdraws the round before it opens. See
  [Cancellation].

Creator

: The account that records a round creation transaction. The creator
  is ordinarily the poll runner, but the vote chain does not
  distinguish it. See [Poll Creation].

Decision

: A voter's chosen option for a proposal, represented as the option's
  0-indexed position in the proposal's option list. A decision is
  committed to in the Vote Commitment and is never published in
  cleartext; see [Proposals and Decisions] and [Vote Reveal Proof].

Decryption threshold ($t$)

: The number of trustees whose partial decryptions are needed to
  decrypt an aggregate ciphertext. Fixed by
  [Election Authority Key Ceremony].

Delegation

: The first phase of the protocol, in which a holder proves ownership
  of notes at the snapshot and transfers voting authority to a
  governance hotkey, producing a Vote Authority Note. See
  [Delegation Phase].

Effective transaction

: A transaction recorded on the vote chain that satisfies every rule
  this ZIP applies to its type, evaluated against the effective
  transactions recorded before it. Only effective transactions
  contribute to a round's derived state. See [Effective Transactions].

Election authority (EA)

: The keypair under which a voting round's vote shares are encrypted
  and whose private key, which exists only as shares held by the
  trustees, decrypts the aggregate tally. A fresh keypair is generated
  for each round by the process in [Election Authority Key Ceremony].

Election authority key ceremony

: The distributed key generation protocol, run over the vote chain, that
  produces a round's election authority public key and each trustee's
  key share. See [Election Authority Key Ceremony].

Final VCT root

: The root of a round's Vote Commitment Tree after the last effective
  delegation or vote transaction recorded at or before the round's vote
  end height. Every Vote Reveal Proof in the round is anchored to it.
  See [Round Lifecycle].

Governance hotkey

: An Orchard-protocol key hierarchy, distinct from the holder's
  spending key, that provides the key material for all voting
  operations after delegation. The hotkey is generated on a
  general-purpose device capable of ZKP construction.

Governance nullifier

: An alternate nullifier (as defined in [^balance-proof]) scoped to the
  governance domain, published during delegation to prevent
  double-delegation of the same note within a voting round.

Ironwood pool

: The Zcash shielded pool over which votes are weighted. The Ironwood
  pool uses the Orchard protocol: its notes, key hierarchy, note
  commitment tree, nullifiers, signatures and proving system are those
  of Orchard as specified in [^protocol]. References in this ZIP to
  Orchard keys, notes, signatures or circuits refer to those
  constructions as used in the Ironwood pool.

Option

: One of the labeled choices a proposal offers. A proposal has between
  2 and $N_{\mathsf{opt}}$ options. See [Proposals and Decisions].

Partial decryption

: A trustee's contribution to decrypting an aggregate ciphertext,
  accompanied by a proof that it was computed with the trustee's share.
  See [Partial Decryption].

Poll runner

: The party that runs a voting round: it chooses the snapshot, names
  the round's trustees, creates the round on the vote chain carrying
  the snapshot's roots, and signs the configuration wallets use to find
  it. Every participant verifies the roots independently; see
  [Snapshot Configuration], [Reading the Snapshot Roots],
  [Poll Creation] and [Poll Signature].

Poll signature

: The poll runner's signature over a round's defining fields, by which
  wallets recognise the round as the one the poll runner is running.
  See [Poll Signature].

Proposal

: A question put to voters in a voting round, with a fixed list of
  options. A round carries between 1 and $\mathsf{MAX}\_\mathsf{PROPOSALS} = 50$
  proposals, identified by 1-indexed sequential integers. See
  [Proposals and Decisions].

Ratification

: A trustee's published statement, made by acknowledging its key share,
  that it has verified the round's snapshot roots, holds a verified
  share for the round, and will take part in its tally. See
  [Ratification].

Reveal window

: The range of vote chain heights, following the voting window, within
  which share reveal transactions are effective: heights above the
  round's $\mathsf{vote}\_\mathsf{end}\_\mathsf{height}$ and at most
  its $\mathsf{reveal}\_\mathsf{end}\_\mathsf{height}$. See
  [Round Lifecycle].

Share commitment

: A blinded Poseidon commitment to one encrypted share's ciphertext.
  The $N_s$ share commitments of a vote are hashed into the shares hash.
  See [Vote Share].

Share nullifier

: A nullifier derived from a vote commitment and share index, published
  when a share is revealed, preventing double-counting.

Shares hash

: The Poseidon hash of a vote's $N_s$ blinded share commitments, bound
  into the Vote Commitment. See [Shares Hash].

Snapshot

: The Zcash mainnet block, identified by height and hash, at which the
  eligible balances of the Ironwood pool are captured for a round. See
  [Snapshot Configuration].

Snapshot height

: The Zcash mainnet block height of the snapshot. See
  [Snapshot Configuration] for constraints.

Submission schedule

: The set of vote chain heights at which a client submits its $N_s$
  share reveal messages, drawn as specified in [Submission Timing].

Trustee

: One of the parties named in a voting round's creation transaction
  that jointly generate its election authority key and each hold a
  share of the private key. Trustees ratify the round and decrypt its
  tally. A trustee holds a trustee account key and a trustee ceremony
  key, and after the ceremony a trustee key share. See
  [Election Authority Key Ceremony] and [Ratification].

Trustee account key

: The Ed25519 keypair under which a trustee signs its vote chain
  transactions. Its public key names the trustee in the round creation
  transaction and identifies its acknowledgement to wallets.

Trustee ceremony key

: A Pallas keypair, whose public key is named in the round creation
  transaction, under which the encrypted shares dealt to the trustee
  during the key ceremony are addressed.

Trustee key share

: $\mathsf{sk}_i$, trustee $i$'s share of the election authority
  private key, produced by the key ceremony and used to compute the
  trustee's partial decryptions.

Trustee share key

: $\mathsf{pk}_i = [\mathsf{sk}_i]\, G$, the public key corresponding
  to trustee $i$'s key share, derivable by anyone from the ceremony's
  published commitments and used to verify the trustee's partial
  decryptions. See [Election Authority Key Ceremony].

Validator

: An operator of the vote chain's consensus. Validators determine which
  transactions are recorded (see [Transaction Inclusion]) and apply
  none of the rules in this ZIP.

VAN nullifier

: A nullifier derived from a VAN commitment and published when the VAN
  is consumed (to cast a vote or delegate), preventing double-spending
  of voting authority.

Vote Authority Note (VAN)

: A commitment inserted into the Vote Commitment Tree that represents
  spendable voting authority. A VAN binds a voting hotkey, a ballot
  count, a voting round identifier, and a proposal authority bitmask.

Vote chain

: The purpose-built chain on which the transactions of a voting round
  are recorded, in order and each at a height. The vote chain records
  transactions and does not validate them against this ZIP; see
  [Vote Chain Record].

Vote Commitment (VC)

: A commitment inserted into the Vote Commitment Tree that binds a
  voter's encrypted share distribution, proposal choice, and vote
  decision for a single proposal. A VC is created when a VAN is consumed
  to cast a vote.

Vote Commitment Tree (VCT)

: An append-only Poseidon Merkle tree, derived per voting round from
  the effective delegation and vote transactions on the vote chain,
  that stores both VANs and VCs as leaves. Trees from different rounds
  are fully isolated. VANs span every proposal in a round, so the tree
  is per round rather than per proposal.

Vote share

: One of $N_s$ encrypted portions of a voter's ballot count within a
  Vote Commitment. Each share is revealed independently for
  homomorphic accumulation.

Voting round

: A bounded period during which a set of proposals are open for voting.
  Each round is identified by a
  $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ derived from its
  snapshot, proposals and deadlines; once its key ceremony completes it
  also has an election authority public key.

Voting window

: The range of vote chain heights within which delegation and vote
  transactions are effective: heights above the height at which the
  round became ACTIVE and at most its
  $\mathsf{vote}\_\mathsf{end}\_\mathsf{height}$. See
  [Round Lifecycle].

# Abstract

This ZIP specifies a shielded voting protocol that allows holders of
notes in the Ironwood pool to cast stake-weighted votes on proposals without revealing
their identity, individual balances, or vote allocations.

The protocol proceeds in three proving phases. First, a *delegation
proof* (building on the Orchard Proof-of-Balance [^balance-proof])
converts proven Ironwood pool balance into a Vote Authority Note on a
purpose-built vote chain. Second, a *vote proof* consumes a VAN to
produce a Vote Commitment containing $N_s$ El Gamal-encrypted shares of
the voter's ballot count, split across vote options. Third, a *vote
reveal proof*, constructed by the voter's client once voting has closed,
opens individual encrypted shares for homomorphic accumulation, without
revealing which Vote Commitment the share originated from or which
option it supports.

After the reveal window closes, anyone can aggregate the revealed
El Gamal ciphertexts per proposal option from the vote chain's record.
The round's trustees — a set disjoint from the validators, among whom
the key was generated without ever existing whole — publish partial
decryptions; anyone combines them via Lagrange interpolation and
verifies the aggregate total. The vote chain records every transaction
and validates none: each rule in this ZIP is applied by verifiers to
the record, and validators determine only what is recorded. The tally
itself reveals only aggregates. What a party holding a threshold of key
shares can recover beyond that, and what the parties controlling block
production can withhold, are stated in [Privacy Implications] and
[Transaction Inclusion] rather than assumed away.

This ZIP also specifies the voting round: how it is anchored to a Zcash
block, how its snapshot roots are read and verified, how the poll
runner signs it, the key ceremony that produces its election authority
key, and the trustee ratification that opens voting. These are rules
any verifier applies to the vote chain's record, not consensus rules of
either chain. The transactions that carry each step, and their
encoding, are specified in [Transaction Formats].


# Motivation

Stake-weighted voting in privacy-preserving systems faces a fundamental
tension: demonstrating voting power requires proving a balance, but
linking that balance to a vote destroys the privacy that shielded
transactions provide.

This ZIP addresses that tension for Zcash's Ironwood shielded pool. The
Orchard Proof-of-Balance [^balance-proof] provides the foundational
primitive, proving note ownership without revealing standard nullifiers.
This ZIP builds on that primitive to specify a complete voting protocol
with the following properties:

- **Unlinkable delegation.** A holder delegates voting power to a
  locally-generated hotkey via a zero-knowledge proof. The delegation
  is unlinkable to the holder's on-chain identity.
- **Private vote splitting.** Votes are decomposed into El Gamal-
  encrypted shares, each revealed independently by the voter's own
  client over its own network connection at its own randomly drawn
  time, so that no party can group a voter's shares by content, timing
  or origin.
- **Homomorphic tallying.** Encrypted shares are recorded on the vote
  chain, and anyone can aggregate the recorded ciphertexts via
  component-wise point addition. Only the aggregate total per proposal
  option is ever decrypted, and no decision appears in cleartext.

The protocol is motivated by coinholder governance in the Zcash
ecosystem, where participants vote on proposals weighted by their ZEC
holdings. The same mechanism applies to any stake-weighted polling system
over a shielded pool built on the Orchard protocol.


# Privacy Implications

**Unlinkability to on-chain identity.** The delegation phase moves
voting authority from the holder's spending key to an unlinkable
governance hotkey. All subsequent voting transactions use this hotkey.
An observer who sees both the governance nullifiers (published during
delegation) and the standard nullifiers (published when notes are later
spent on-chain) cannot link them without knowledge of $\mathsf{nk}$.
This follows from the alternate nullifier unlinkability property
established in [^balance-proof].

**Share unlinkability.** A share reveal transaction exposes a share
nullifier, a vector of El Gamal ciphertexts, a proposal identifier, the
round's final VCT root and the round identifier. The nullifier is a
Poseidon hash whose preimage includes the private vote commitment and a
private blind factor; each ciphertext carries fresh randomness; the
root and round identifier take the same value in every reveal of the
round. Nothing in the transaction's contents identifies the vote
commitment it opens or relates it to any other reveal. The anonymity
set of a revealed share is every share revealed for the same proposal.
This holds against a chain observer and against a coalition holding
$t$ key shares: such a coalition can decrypt any individual share, but
decryption yields a share's value, not an association with other
shares. Whether two shares belong to one vote is not recoverable from
any content the protocol publishes. What remains is metadata — the
height at which each reveal is submitted and the network path it
arrives by — which [Share Submission] and [Submission Timing] address.

**Balance hiding via vote splitting.** A voter's ballot count is
decomposed into $N_s$ shares, each encrypted under the election
authority's public key and revealed independently. A party that
decrypts one share learns an estimate of the voter's total whose
accuracy is bounded by the decomposition (see [Vote Share] and
[Why Randomized Share Decomposition]); a party that gathers and
decrypts several shares of one vote estimates the total more closely,
and one that gathers all $N_s$ recovers it exactly. Splitting therefore
protects amounts only insofar as shares cannot be grouped. The protocol
publishes nothing that groups them (see the previous paragraph), so the
submission process carries the remaining burden: no two shares of one
vote may be linkable by timing, by network origin, or by any metadata a
vote chain node or validator logs. That is the purpose of the per-share
network isolation and memoryless scheduling required in
[Share Submission] and [Submission Timing], and the reason no party
other than the voter's client ever holds a share reveal message before
it is recorded.

**Individual vote amounts hidden from the public.** Each share is an
El Gamal ciphertext whose plaintext value is never revealed on-chain,
and only the aggregate total per (proposal, option) pair is decrypted
at tally time. This holds against any party that does not hold $t$
shares of $\mathsf{ea}\_\mathsf{sk}$. A coalition holding $t$ shares can
open any individual share ciphertext, and those ciphertexts are
recorded on the vote chain permanently; what such a coalition then
holds is $N_s$ unlinkable fragments per voter, mixed among every other
voter's fragments for the same proposal.

**Decision secrecy.** A voter's decision is a private witness to both
the Vote Proof and the Vote Reveal Proof and appears in no transaction.
Each reveal carries one ciphertext per option position; the position
that encrypts the share value is indistinguishable from the positions
that encrypt zero without $t$ key shares. Chain observers and
validators therefore learn neither how any share voted nor the
per-option totals while a round is open. A coalition holding $t$ shares
that decrypts an individual share learns that share's option along with
its value, subject to the unlinkability above.

**Vote commitment unlinkability.** The Vote Reveal Proof proves
that a revealed share belongs to some valid Vote Commitment in the VCT
without revealing which one. Blinded per-share commitments prevent
observers from recomputing $\mathsf{shares}\_\mathsf{hash}$ from on-chain
ciphertexts and linking revealed shares back to a specific VC. Every
reveal in a round is anchored to the same final VCT root, so the anchor
partitions nothing.

**Trust assumptions.** The election authority private key is never
held by any party. The key ceremony is a distributed key generation
among the round's trustees (see
[Election Authority Key Ceremony]); each trustee ends the ceremony
with a share and nobody, at any point, with the key. An adversary must
obtain at least $t$ shares to decrypt an individual share ciphertext,
where $t$ and the trustee set are fixed by the ceremony and published
with the round. Vote splitting does not substitute for that threshold: it bounds
what a coalition that does reach $t$ can learn about any one voter, by
ensuring that what it can decrypt cannot be grouped to determine voter
balances.

No party other than the voter's client holds a share reveal message
before it is recorded. The vote chain node that receives a message
learns what a chain observer learns from the same message once it is
recorded, plus the network origin of that one submission, which is why
[Share Submission] requires an independent network path per
submission. The vote chain itself is trusted for inclusion only: its
validators record transactions and apply none of this ZIP's rules, and
what they can do by choosing what to record is stated in
[Transaction Inclusion].

**Non-membership tree queries.** Obtaining exclusion proofs for the
nullifier non-membership tree during delegation requires a source for
that tree's leaves. A client that holds the nullifier set at the
snapshot height constructs its own exclusion proof and reveals nothing.
A client that queries a server for one reveals which nullifier it asked
about, and therefore which note it holds, unless that server's retrieval
protocol conceals the query, as private information retrieval (PIR)
does. Such a protocol is specified in
`draft-valargroup-nullifier-pir` [^nullifier-pir]; whether a deployment
offers one is a deployment matter (see [Non-requirements] and the
nullifier service step of [Snapshot Configuration]).


# Requirements

- A holder's on-chain identity (spending key, standard nullifiers) is
  not linkable to their voting activity.
- The number of user-facing signatures required for delegation is
  minimized.
- No double-delegation for the same note within a voting round.
- No double voting for the same voting share within the same proposal.
- Individual vote amounts are not revealed at any point; only aggregate
  totals per (proposal, option) pair are recoverable. A party holding
  $t$ key shares can decrypt an individual share ciphertext, but no
  party can determine which shares belong to one vote, so no voter's
  total is recoverable; see [Privacy Implications].
- A voter's decision is not revealed to any party holding fewer than
  $t$ key shares.
- The aggregate tally is publicly verifiable: any party can recompute
  the aggregation and check the decryption from the vote chain's
  record.
- Every rule in this ZIP can be applied to the vote chain's record by
  any party, and no integrity property of a round depends on the vote
  chain's validators applying any of them.
- No party that takes part in the vote chain's consensus holds a share
  of any round's election authority key.
- Anyone may run a vote chain node that accepts transactions and
  forwards them to validators, and a client may submit to any node,
  including its own.
- The delegation phase is compatible with hardware wallets that support
  only the standard Orchard PCZT [^pczt] signing flow, without
  requiring firmware changes specific to the voting protocol.


# Non-requirements

- The vote chain's consensus engine, block structure, transaction
  envelope and node API. The transactions this ZIP defines, their
  encoding, and the rules by which any party interprets them are in
  scope; see [Transaction Formats] and [Vote Chain Record].
- How operators are organised to run a deployment: roles, validator
  onboarding, nullifier service operation, deployment architecture, and
  audit procedures. These are specified in
  `draft-valargroup-shielded-voting-setup` [^voting-setup].
- Post-quantum security of the El Gamal encryption layer is out of
  scope.
- Retrieval of nullifier non-membership proofs by clients that do not
  hold the nullifier set. The tree itself is specified in
  [^balance-proof]; a retrieval protocol that conceals which nullifier
  a client asks about is specified in
  `draft-valargroup-nullifier-pir` [^nullifier-pir]; whether a
  deployment offers one is a deployment concern.

# High-level summary

This section is non-normative.

The protocol proceeds in four phases within a voting round. Every
transaction below is recorded on the vote chain; the vote chain
validates none of them, and each party that reads the record applies
the rules of this ZIP to decide which recorded transactions count (see
[Vote Chain Record]).

**Phase 1: Delegation.** A holder proves ownership of unspent notes in
the Ironwood pool at a pool snapshot (using the Claim circuit from
[^balance-proof]) and delegates voting authority to a locally-generated
governance hotkey. The delegation produces a Vote Authority Note (VAN)
that is inserted into the Vote Commitment Tree on the vote chain.
Governance nullifiers are published to prevent double-delegation.

**Phase 2: Voting.** The governance hotkey consumes a VAN by publishing
its VAN nullifier, and produces two new VCT leaves: a replacement VAN
with the voted proposal's authority bit cleared, and a Vote Commitment
containing $N_s$ El Gamal-encrypted shares of the voter's ballot count.
Delegation and voting both take place during the voting window.

**Phase 3: Share reveal.** Once the voting window closes, the VCT is
frozen and its final root is fixed. The voter's client constructs a
Vote Reveal Proof for each of its $N_s$ shares against that root. Each
proof opens one share as a vector of ciphertexts, one per option
position, without revealing which Vote Commitment it came from or which
position carries the share's value. The client itself submits each
resulting message to the vote chain over an independent network path
at an independently drawn height within the reveal window, coming
online to do so; no other party submits on its behalf.

**Phase 4: Tally.** After the reveal window closes, anyone can
aggregate the recorded ciphertext vectors per (proposal, option) pair.
At least $t$ trustees publish partial decryptions of each aggregate
ciphertext, and anyone combines them via Lagrange interpolation to
recover the total ballot count (via the bounded discrete-log recovery
procedure defined in [Decryption]). Correctness is publicly verifiable:
anyone can recompute the aggregation and the Lagrange combination from
the recorded reveals and partial decryptions.


# Specification

## El Gamal Encryption on Pallas

The protocol uses additively homomorphic El Gamal encryption over the
Pallas curve to encrypt vote share amounts.

### Setup

Let $G$ be the Pallas $\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$
generator [^protocol-concretespendauthsig], and let $\mathbb{F}_q$ denote
the scalar field of the Pallas curve [^protocol-pallasandvesta]. The
election authority keypair for a voting round is:

- $\mathsf{ea}\_\mathsf{sk} \in \mathbb{F}_q$ — a scalar sampled
  uniformly at random;
- $\mathsf{ea}\_\mathsf{pk} = [\mathsf{ea}\_\mathsf{sk}]\, G$ — the
  corresponding public key.

Every El Gamal and ECIES operation specified in this ZIP MUST use this
generator. Using any other point would break the additive homomorphism
that [Tally] depends on, and would make ciphertexts incompatible with
the [Vote Reveal Proof] circuit.

The keypair is generated afresh for each round by
[Election Authority Key Ceremony], which also fixes how
$\mathsf{ea}\_\mathsf{sk}$ is shared among the round's trustees and
the threshold $t$ used in [Tally].

### Encryption

To encrypt a ballot count $v$ (see [Tally units]) under randomness
$r \leftarrow \mathbb{F}_q$:

$$\mathsf{Enc}(v, r) = \bigl([r]\, G,\; [v]\, G + [r]\, \mathsf{ea}\_\mathsf{pk}\bigr)$$

The ciphertext is a pair of Pallas points $(C_1, C_2)$. The randomness
$r$ MUST be sampled with a CSPRNG and MUST NOT be reused across
ciphertexts.

### Additive Homomorphism

Component-wise point addition of two ciphertexts yields a valid
encryption of the sum of their plaintexts:

$$\mathsf{Enc}(a, r_1) + \mathsf{Enc}(b, r_2) = \mathsf{Enc}(a + b,\; r_1 + r_2)$$

This is what allows any party to aggregate the share ciphertexts
recorded for a round into a single ciphertext per
$(\mathsf{proposal}\_\mathsf{id}, j)$ pair, for each option position
$j$, without decrypting any of them (see [Aggregation]), and it is why
no party needs $\mathsf{ea}\_\mathsf{sk}$ before the reveal window
closes.

### Decryption

Given an aggregate ciphertext $(C_{1,\mathsf{agg}}, C_{2,\mathsf{agg}})$
and $\mathsf{ea}\_\mathsf{sk}$:

$$C_{2,\mathsf{agg}} - [\mathsf{ea}\_\mathsf{sk}]\, C_{1,\mathsf{agg}} = [\mathsf{total}\_\mathsf{value}]\, G$$

Recovering $\mathsf{total}\_\mathsf{value}$ from
$[\mathsf{total}\_\mathsf{value}]\, G$ requires a bounded discrete
logarithm search, for which baby-step giant-step is sufficient: the
plaintext is a ballot count, bounded above by the total ZEC supply
divided by the ballot unit (see [Tally units]).

This operation is never performed with a reconstructed
$\mathsf{ea}\_\mathsf{sk}$ in normal operation; [Tally] specifies the
threshold procedure that computes
$[\mathsf{ea}\_\mathsf{sk}]\, C_{1,\mathsf{agg}}$ without any party
holding the secret key.


## Chaum-Pedersen DLEQ Proofs

Correct use of secret key material during threshold decryption is
demonstrated with a non-interactive Chaum-Pedersen proof of
discrete-logarithm equality [^chaum-pedersen], instantiated over Pallas
with a Fiat-Shamir challenge derived from BLAKE2b-256 [^blake2].

**Statement.** Given Pallas point pairs $(G, P)$ and $(H, Q)$, a DLEQ
proof demonstrates $\log_G(P) = \log_H(Q)$: that a single scalar $x$
satisfies both $P = [x]\, G$ and $Q = [x]\, H$.

A proof is a pair of Pallas scalars $(e, z)$, serialized as 64 bytes
($e \mathbin\| z$, 32 bytes each, each a canonical little-endian
encoding of an element of $\mathbb{F}_q$).

### Challenge Derivation

$$e = \mathsf{HashToScalar}\bigl(\texttt{"svote-dleq-v1"} \mathbin\| \mathsf{repr}(G) \mathbin\| \mathsf{repr}(P) \mathbin\| \mathsf{repr}(H) \mathbin\| \mathsf{repr}(Q) \mathbin\| \mathsf{repr}(R_1) \mathbin\| \mathsf{repr}(R_2)\bigr)$$

where $\mathsf{repr}$ is the 32-byte compressed affine encoding of a
Pallas point and $\mathsf{HashToScalar}$ applies unkeyed BLAKE2b-256
to the concatenation and maps the resulting 32-byte digest to an
element of $\mathbb{F}_q$. The domain separator
$\texttt{"svote-dleq-v1"}$ (13 bytes, ASCII) prevents challenges from
being reused across protocols.

### Proof Generation

A prover holding $x$ computes:

1. Sample $k \leftarrow \mathbb{F}_q$ uniformly at random, using a
   CSPRNG.
2. $R_1 = [k]\, G$ and $R_2 = [k]\, H$.
3. $e = \mathsf{DLEQChallenge}(G, P, H, Q, R_1, R_2)$ per
   [Challenge Derivation].
4. $z = k + e \cdot x$.
5. Output $(e, z)$.

### Proof Verification

A verifier given $(G, P, H, Q)$ and a proof $(e, z)$ MUST:

1. Parse $e$ and $z$ as canonical elements of $\mathbb{F}_q$, rejecting
   any non-canonical encoding, and validate $G$, $P$, $H$ and $Q$ as
   points on the Pallas curve.
2. Compute $R_1 = [z]\, G - [e]\, P$ and $R_2 = [z]\, H - [e]\, Q$.
3. Compute $e' = \mathsf{DLEQChallenge}(G, P, H, Q, R_1, R_2)$.
4. Accept if and only if $e' = e$.


## ECIES on Pallas

Distribution of key shares to trustees uses ECIES [^ecies]
instantiated on Pallas:

- **Key encapsulation**: ephemeral Diffie-Hellman on Pallas with
  generator $G$.
- **Key derivation**: $k = \mathsf{SHA256}\bigl(\mathsf{repr}(E) \mathbin\| \mathsf{x}(S)\bigr)$,
  where $E$ is the ephemeral public key, $S$ the shared secret point,
  $\mathsf{repr}$ the 32-byte compressed encoding, and $\mathsf{x}(S)$
  the $x$-coordinate obtained by taking that encoding and clearing
  bit 7 of byte 31 (the sign bit).
- **Symmetric encryption**: ChaCha20-Poly1305 with an all-zero nonce.

A fresh ephemeral scalar MUST be generated for each recipient; reusing
one across recipients would correlate their encapsulations and, with
the zero nonce, would reuse a symmetric key across messages. The
recipient MUST verify its decrypted share against the sender's
published Feldman commitments, as specified in Stage 3 of
[Election Authority Key Ceremony].


## Data Structures

### Voting Round Identifier

Each voting round is identified by a 32-byte
$\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$, derived
deterministically from the fields of its round creation transaction
(see [Poll Creation]) by Poseidon hashing. The identifier is not
carried in that transaction; every party computes it from the same
inputs, and every later transaction of the round names it.

$$\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}} = \mathsf{Poseidon}\bigl(\mathsf{snapshot}\_\mathsf{height},\ \mathsf{bh}\_\mathsf{lo},\ \mathsf{bh}\_\mathsf{hi},\ \mathsf{ph}\_\mathsf{lo},\ \mathsf{ph}\_\mathsf{hi},\ \mathsf{vote}\_\mathsf{end}\_\mathsf{height},\ \mathsf{reveal}\_\mathsf{end}\_\mathsf{height},\ \mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root},\ \mathsf{nc}\_\mathsf{root},\ \mathsf{th}\_\mathsf{lo},\ \mathsf{th}\_\mathsf{hi},\ \mathsf{cp}\_\mathsf{lo},\ \mathsf{cp}\_\mathsf{hi}\bigr)$$

The hash uses the $\mathsf{ConstantLength}\langle 13 \rangle$ variant
of the Poseidon instantiation specified in [Poseidon Instantiation].

where:

- $\mathsf{snapshot}\_\mathsf{height} \in \{ 0 .. 2^{64}-1 \}$ — the
  Zcash mainnet block height of the chosen snapshot, encoded as a
  Pallas base field element.
- $\mathsf{bh}\_\mathsf{lo}, \mathsf{bh}\_\mathsf{hi} \in \{ 0 .. 2^{128}-1 \}$
  — the low and high 128-bit halves of
  $\mathsf{snapshot}\_\mathsf{blockhash}$ in little-endian byte
  order, each encoded as a Pallas base field element.
- $\mathsf{ph}\_\mathsf{lo}, \mathsf{ph}\_\mathsf{hi} \in \{ 0 .. 2^{128}-1 \}$
  — the low and high 128-bit halves of $\mathsf{proposals}\_\mathsf{hash}$
  (see [Proposals Hash]) in little-endian byte order.
- $\mathsf{vote}\_\mathsf{end}\_\mathsf{height} \in \{ 0 .. 2^{64}-1 \}$
  — the vote chain block height after which delegation and vote
  transactions are no longer effective, encoded as a Pallas base field
  element.
- $\mathsf{reveal}\_\mathsf{end}\_\mathsf{height} \in \{ 0 .. 2^{64}-1 \}$
  — the vote chain block height after which share reveal transactions
  are no longer effective, encoded as a Pallas base field element.
- $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root} \in \{ 0 .. q_{\mathbb{P}}-1 \}$
  — root of the nullifier non-membership IMT at the snapshot
  height. MUST be a canonical Pallas base field element.
- $\mathsf{nc}\_\mathsf{root} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ —
  Ironwood pool note commitment tree root at the snapshot height.
  MUST be a canonical Pallas base field element.
- $\mathsf{th}\_\mathsf{lo}, \mathsf{th}\_\mathsf{hi} \in \{ 0 .. 2^{128}-1 \}$
  — the low and high 128-bit halves of $\mathsf{trustees}\_\mathsf{hash}$
  (see [Trustees Hash]) in little-endian byte order.
- $\mathsf{cp}\_\mathsf{lo}, \mathsf{cp}\_\mathsf{hi} \in \{ 0 .. 2^{128}-1 \}$
  — the low and high 128-bit halves of the creator's public key
  $\mathsf{creator}\_\mathsf{pk}$ (see [Poll Creation]) in little-endian
  byte order.

$\mathsf{snapshot}\_\mathsf{blockhash}$, $\mathsf{proposals}\_\mathsf{hash}$,
$\mathsf{trustees}\_\mathsf{hash}$ and $\mathsf{creator}\_\mathsf{pk}$
are split into two 128-bit limbs because they are not necessarily
canonical Pallas field elements. $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ and
$\mathsf{nc}\_\mathsf{root}$ are themselves Poseidon-derived in
their respective trees and are therefore already canonical.

The identifier binds everything that defines a round: its snapshot,
its proposals, its deadlines, its trustees and its creator. Two round
creation transactions with the same identifier describe the same round,
and only the first recorded can be effective (see [Poll Creation]). The
identifier does not depend on the election authority public key, which
does not exist when the round is created; the key ceremony binds to the
identifier, not the reverse (see [Election Authority Key Ceremony]).

### Vote Authority Note (VAN)

A VAN represents spendable voting authority on the vote chain. Its
commitment is computed in two layers:

$$\mathsf{van}\_\mathsf{core} = \mathsf{Poseidon}\bigl(\mathsf{DOMAIN}\_\mathsf{VAN}, \mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}}, \mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}}, \mathsf{num}\_\mathsf{ballots}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{proposal}\_\mathsf{authority}\bigr)$$

$$\mathsf{van} = \mathsf{Poseidon}\bigl(\mathsf{van}\_\mathsf{core}, \mathsf{gov}\_{\mathsf{comm}\_\mathsf{rand}}\bigr)$$

The first layer binds the structural fields; the second layer blinds
the commitment with randomness. These are two separate Poseidon
invocations ($\mathsf{ConstantLength}\langle 6 \rangle$ then
$\mathsf{ConstantLength}\langle 2 \rangle$), not a single 7-input
sponge absorption.

where:

- $\mathsf{DOMAIN}\_\mathsf{VAN} = 0$ — domain tag distinguishing VANs from VCs
  in the shared VCT.
- $\mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ —
  $\mathsf{Extract}\_{\mathbb{P}}$ of the diversified base of the governance
  hotkey address.
- $\mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ —
  $\mathsf{Extract}\_{\mathbb{P}}$ of the diversified transmission key of the
  governance hotkey address.
- $\mathsf{num}\_\mathsf{ballots} \in \{1 \ldots 2^{30}\}$ — total voting
  weight in ballots.
- $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ — scopes this VAN to a specific voting round.
- $\mathsf{proposal}\_\mathsf{authority} \in \{0 \ldots 2^{51}-1\}$ — bitmask
  encoding which proposals this VAN is authorized to vote on.
  $\mathsf{proposal}\_\mathsf{id}$ values 1–50 map to bits 1–50; bit 0 is
  reserved (see [Why Proposal Identifiers Start at 1]). Full authority
  is $\mathsf{MAX}\_{\mathsf{PROPOSAL}\_\mathsf{AUTHORITY}} = 2^{51} - 1$.
- $\mathsf{gov}\_{\mathsf{comm}\_\mathsf{rand}} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ —
  commitment randomness.

**VAN nullifier.** When a VAN is consumed (to vote or delegate), its
nullifier is:

$$\mathsf{van}\_\mathsf{nullifier} = \mathsf{Poseidon}\bigl(\mathsf{vsk.nk}, \mathsf{tag}_{\mathsf{van}}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{van}\bigr)$$

where $\mathsf{tag}_{\mathsf{van}}$ is the field-element encoding of the domain
separator `"vote authority spend"` and $\mathsf{vsk.nk}$ is the nullifier
deriving key from the governance hotkey's full viewing key.

A VAN MUST be created during delegation (Phase 1) and
consumed during voting (Phase 2), which MUST produce a replacement VAN
with updated $\mathsf{proposal}\_\mathsf{authority}$.

The VAN model is
designed to support future extensions such as partial delegation
(splitting $\mathsf{num}\_\mathsf{ballots}$ across multiple delegates),
but this ZIP specifies only the delegation and voting operations.

### Vote Commitment (VC)

A VC commits to a vote on a specific proposal:

$$\mathsf{vc} = \mathsf{Poseidon}\bigl(\mathsf{DOMAIN}\_\mathsf{VC}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{shares}\_\mathsf{hash}, \mathsf{proposal}\_\mathsf{id}, \mathsf{vote}\_\mathsf{decision}\bigr)$$

where:

- $\mathsf{DOMAIN}\_\mathsf{VC} = 1$ — domain tag.
- $\mathsf{shares}\_\mathsf{hash} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ — a hash
  over blinded commitments to all $N_s$ encrypted shares (see
  [Shares Hash]).
- $\mathsf{proposal}\_\mathsf{id} \in \{1 \ldots \mathsf{MAX}\_\mathsf{PROPOSALS}\}$ —
  which proposal this vote targets.
- $\mathsf{vote}\_\mathsf{decision} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ — the
  voter's choice (0-indexed into the proposal's declared options).

A VC MUST be created during voting (Phase 2) and opened during share reveal (Phase 3).

The VC hash is a public input of the Vote Proof, carried in the vote
transaction, and is inserted into the VCT when that transaction is
effective.

Its preimage fields
($\mathsf{shares}\_\mathsf{hash}$ and $\mathsf{vote}\_\mathsf{decision}$)
MUST be private witnesses in that proof.

During share reveal, the Vote
Reveal Proof MUST prove membership in the VCT without exposing which VC
is being opened, and MUST NOT expose $\mathsf{vote}\_\mathsf{decision}$.

### Vote Share

A vote share is one of $N_s$ encrypted portions of a voter's ballot
count within a VC. $N_s$ is a protocol parameter (see
[Protocol parameters]) fixed by the circuits: the Vote Proof encrypts
exactly $N_s$ values and the Vote Reveal Proof opens one of $N_s$. A
decomposition therefore always fills $N_s$ slots; a slot MAY hold the
value zero.

The shares MUST sum to $\mathsf{num}\_\mathsf{ballots}$ and each share
MUST be in $[0, 2^{30})$. In addition, the decomposition MUST satisfy
the following requirement.

**Decomposition requirement.** The share values MUST NOT be a
deterministic function of $\mathsf{num}\_\mathsf{ballots}$. Concretely,
a client MUST NOT divide the ballot count evenly across the $N_s$
shares, and MUST NOT use any other rule under which the value of a
single share determines $\mathsf{num}\_\mathsf{ballots}$ up to a
publicly known bound.

A conforming default decomposition is:

1. Sample $N_s - 1$ values $u_1 \ldots u_{N_s - 1}$ uniformly at random
   from $\{0 \ldots \mathsf{num}\_\mathsf{ballots}\}$.
2. Sort them, and set the share values to the successive differences of
   $0, u_{(1)}, \ldots, u_{(N_s - 1)}, \mathsf{num}\_\mathsf{ballots}$.
3. Apply a uniformly random permutation to the resulting $N_s$ values
   before assigning them to share indices.

See [Why Randomized Share Decomposition] for what this does and does not
achieve; in particular, it reduces but does not eliminate the
information a decrypted share carries about the voter's total.

For share index $i \in \{0 \ldots N_s - 1\}$:

- $\mathsf{v}\_\mathsf{i} \in \{0 \ldots 2^{30} - 1\}$ — plaintext share
  amount in ballots (private, never revealed).
- $r_i$ — El Gamal encryption randomness (Pallas scalar, private).
- $\mathsf{blind}\_\mathsf{i} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ — per-share
  blind factor for the blinded share commitment (independent of $r_i$).
- $\mathsf{enc}\_{\mathsf{share}\_\mathsf{i}} = \mathsf{Enc}(\mathsf{v}\_\mathsf{i}, r_i) = (C_{1,i}, C_{2,i})$
  — El Gamal ciphertext.

**Blinded share commitment:**

$$\mathsf{share}\_{\mathsf{comm}\_\mathsf{i}} = \mathsf{Poseidon}(\mathsf{blind}\_\mathsf{i}, C_{1,i,x}, C_{2,i,x}, C_{1,i,y}, C_{2,i,y})$$

where $C_{1,i,x}$, $C_{2,i,x}$ denote the $x$-coordinates and
$C_{1,i,y}$, $C_{2,i,y}$ denote the $y$-coordinates of the ciphertext
points. See [Why Share Commitments Bind Full Curve Points].

### Shares Hash

The shares hash aggregates all $N_s$ blinded share commitments:

$$\mathsf{shares}\_\mathsf{hash} = \mathsf{Poseidon}(\mathsf{share}\_{\mathsf{comm}\_\mathsf{0}}, \mathsf{share}\_{\mathsf{comm}\_\mathsf{1}}, \ldots, \mathsf{share}\_{\mathsf{comm}_{N_s - 1}})$$

This is an $N_s$-input Poseidon sponge hash.

### Share Nullifier

When a share is revealed, its nullifier is:

$$\mathsf{share}\_\mathsf{nullifier} = \mathsf{Poseidon}\bigl(\mathsf{tag}_{\mathsf{share}}, \mathsf{vc}, \mathsf{share}\_\mathsf{index}, \mathsf{blind}\bigr)$$

where $\mathsf{tag}_{\mathsf{share}}$ is the field-element encoding of
`"share spend"`, $\mathsf{vc}$ is the vote commitment (private), and
$\mathsf{blind}$ is the blind factor for the revealed share.

The share nullifier is a public input of the Vote Reveal Proof,
carried in the share reveal transaction; a share reveal whose
nullifier is already in the round's share nullifier set is not
effective (see [Nullifier Sets]), which prevents double-counting.

### Vote Commitment Tree

Each party that interprets the vote chain's record derives, for each
voting round, a Vote Commitment Tree: an incremental Merkle tree
[^protocol-merkletree] of depth $\mathsf{MerkleDepth}^{\mathsf{vct}} = 24$
that stores both VANs and VCs as leaves. Leaves, roots, and tree state
from one round MUST NOT carry over into another. The tree MUST use the
same append-only data structure as the Orchard note commitment tree,
but with Poseidon over the Pallas scalar field for internal node
hashing instead of Sinsemilla (see [Poseidon Instantiation]).

Domain separation between VANs and VCs is achieved structurally: the
first Poseidon input is $\mathsf{DOMAIN}\_\mathsf{VAN} = 0$ for VANs and
$\mathsf{DOMAIN}\_\mathsf{VC} = 1$ for VCs, making it impossible for a valid VAN
preimage to produce the same hash as a valid VC preimage.

Leaves MUST be inserted in record order from effective transactions
only (see [Effective Transactions]): an effective delegation
transaction inserts one VAN; an effective vote transaction inserts a
new VAN and then a VC. A transaction that is not effective inserts
nothing. Because the record and the rules are the same for every
party, every party derives the same tree, and the tree's root after
any height is a well-defined value that proofs can anchor to.

### Nullifier Sets

For each voting round, three disjoint nullifier sets are derived from
the record:

1. **Governance nullifiers**: prevent double-delegation of mainchain
   Ironwood pool notes within a voting round.
2. **VAN nullifiers**: prevent double-spending of voting authority.
3. **Share nullifiers**: prevent double-counting of revealed shares.

Each set is append-only within a voting round and contains the
nullifiers published by the round's effective transactions, in record
order. A transaction that publishes a nullifier already present in the
corresponding set is not effective, and adds nothing to any set (see
[Effective Transactions]). Where two recorded transactions publish the
same nullifier, the earlier one in the record is the one that can be
effective.

The protocol defines three tags:

| Tag constant | String | Length |
|---|---|---|
| $\mathsf{tag}\_{\mathsf{gov}}$ | `"governance authorization"` | 24 bytes |
| $\mathsf{tag}\_{\mathsf{van}}$ | `"vote authority spend"` | 20 bytes |
| $\mathsf{tag}\_{\mathsf{share}}$ | `"share spend"` | 11 bytes |

## Governance Hotkey

The governance hotkey is a separate Orchard key hierarchy generated on
a general-purpose device (e.g., a mobile phone). It is distinct from
the holder's Orchard spending key, which may reside on a hardware
wallet. The hotkey provides the key material for all voting operations
after delegation.

A governance hotkey consists of:

- A fresh spend-authorizing key $\mathsf{vsk}$, from which
  $\mathsf{vsk.ak} = [\mathsf{vsk}]\, G$ is derived.
- A nullifier deriving key $\mathsf{vsk.nk}$, derived from the hotkey's
  spending key via the standard Orchard key hierarchy.
- A CommitIvk trapdoor $\mathsf{rivk}\_\mathsf{v}$, derived from the
  hotkey's spending key.
- A diversified address at a chosen diversifier index, consisting of
  the diversified base point and the diversified transmission key point,
  where the transmission key satisfies
  $[\mathsf{ivk}\_\mathsf{v}]\, \mathsf{g}\_{\mathsf{d}}$
  with $\mathsf{ivk}\_\mathsf{v} = \mathsf{CommitIvk}\_{\mathsf{rivk}\_\mathsf{v}}\!\bigl(\mathsf{Extract}\_{\mathbb{P}}(\mathsf{vsk.ak}),\; \mathsf{vsk.nk}\bigr)$.
  The VAN commitment (see [Vote Authority Note (VAN)]) stores the
  $x$-coordinates $\mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}} = \mathsf{Extract}\_{\mathbb{P}}(\mathsf{g}\_\mathsf{d})$
  and $\mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}} = \mathsf{Extract}\_{\mathbb{P}}(\mathsf{pk}\_\mathsf{d})$;
  the full curve points are used in the diversified address integrity
  check (see [Vote Proof]).

The Delegation Proof does not constrain the hotkey address to match the
holder's key; the output address is bound to the delegation
transitively through the VAN commitment and the rho binding (see
[Delegation Proof]), which the holder's hardware wallet authenticates
via the spend authorization signature.

The hotkey MUST be derived deterministically from a seed
$\mathsf{seed}\_\mathsf{v}$. The wallet MUST generate
$\mathsf{seed}\_\mathsf{v}$ by creating a fresh BIP 39
mnemonic [^bip39], converting it to a 64-byte BIP 39 seed, and
storing the mnemonic in local secure storage (e.g., the platform
keychain). The mnemonic MUST be generated independently of the
holder's wallet mnemonic and MUST NOT be exported or backed up;
it is needed only for the duration of the voting round.

The wallet computes

$$\mathsf{sk}\_\mathsf{v} = \mathsf{Blake2b}\text{-}512(\texttt{"ZcashVotingHotKy"},\; \mathsf{seed}\_\mathsf{v})$$

interpreted as a Pallas scalar via $\mathsf{FromUniformBytes}$, and
then derives $\mathsf{vsk.ak}$, $\mathsf{vsk.nk}$, and
$\mathsf{rivk}\_\mathsf{v}$ from $\mathsf{sk}\_\mathsf{v}$ following
§ 4.2.3 'Orchard Key Components' [^protocol-orchardkeycomponents].
See [Why Deterministic Hotkey Derivation].

### Per-Vote Secret Derivation

The El Gamal encryption randomness $r_i$, per-share blind factors
$\mathsf{blind}\_\mathsf{i}$, and spend authorization randomizer
$\alpha_v$ used in the Vote Proof MUST be derived deterministically
from the hotkey seed. This ensures that all per-vote secrets can be
reconstructed from the BIP 39 mnemonic if the app is terminated
between delegation and voting.

For a vote on proposal $\mathsf{proposal}\_\mathsf{id}$ consuming
VAN commitment $\mathsf{van}\_\mathsf{old}$ in voting round
$\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$, the wallet MUST
derive a 64-byte per-vote root:

$$\mathsf{vote}\_\mathsf{root} = \mathsf{Blake2b}\text{-}512\bigl(\texttt{"ZcashVoteSecret"},\; \mathsf{sk}\_\mathsf{v} \| \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}} \| \mathsf{proposal}\_\mathsf{id} \| \mathsf{van}\_\mathsf{old}\bigr)$$

where $\|$ denotes concatenation of canonical little-endian encodings
(32 bytes for field elements, 1 byte for
$\mathsf{proposal}\_\mathsf{id}$).

From $\mathsf{vote}\_\mathsf{root}$, the wallet MUST derive each
per-share secret:

$$r_i = \mathsf{FromUniformBytes}\!\bigl(\mathsf{Blake2b}\text{-}512(\texttt{"ZcashVoteElGam"},\; \mathsf{vote}\_\mathsf{root} \| i)\bigr)$$

$$\mathsf{blind}\_\mathsf{i} = \mathsf{FromUniformBytes}\!\bigl(\mathsf{Blake2b}\text{-}512(\texttt{"ZcashVoteBlind"},\; \mathsf{vote}\_\mathsf{root} \| i)\bigr)$$

where $i$ is the share index encoded as a single byte. The spend
authorization randomizer is:

$$\alpha_v = \mathsf{FromUniformBytes}\!\bigl(\mathsf{Blake2b}\text{-}512(\texttt{"ZcashVoteAlpha"},\; \mathsf{vote}\_\mathsf{root})\bigr)$$

All Blake2b invocations above use the first argument as the
personalization parameter (truncated or padded to 16 bytes per the
Blake2b spec) and the second argument as the input message.
$\mathsf{FromUniformBytes}$ interprets a 64-byte input as a Pallas
scalar. [^protocol-pallasandvesta]

See [Why Deterministic Hotkey Derivation].


## Ballot Scaling

Note values are denominated in zatoshi. The Delegation Proof
MUST convert zatoshi to ballots:

$$\mathsf{num}\_\mathsf{ballots} = \left\lfloor \frac{\sum v_i}{12{,}500{,}000} \right\rfloor$$

where $v_i$ are the values of the delegated notes. One ballot
equals 0.125 ZEC.

The prover MUST witness $\mathsf{num}\_\mathsf{ballots}$ and a
remainder $r$. The Delegation Proof circuit MUST enforce:

1. $\mathsf{num}\_\mathsf{ballots} \times 12{,}500{,}000 + r = \sum v_i$
2. $0 \leq r < 2^{24}$ (see [Why a 24-bit Remainder Range])
3. $\mathsf{num}\_\mathsf{ballots} \geq 1$
4. $\mathsf{num}\_\mathsf{ballots} \leq 2^{30}$

The 30-bit upper bound accommodates up to $\approx 134$ million ZEC,
well above the 21 million ZEC supply cap. The minimum of 1 ballot
ensures that holdings below 0.125 ZEC MUST NOT produce voting
authority.

Because one ballot equals 0.125 ZEC, the unit of the tally is the
ballot, not ZEC. Any participation threshold, quorum, or published
result expressed in ZEC MUST be converted before it is compared against
a tally: a threshold of 1,000,000 ZEC corresponds to 8,000,000 ballots.
Implementations and operational procedures MUST state which unit a
published figure is in.


## Delegation Phase

### Delegation Proof

The Delegation Proof establishes that a holder owns unspent Ironwood
pool notes at a pool snapshot and converts the proven balance into a VAN on
the vote chain.

The Delegation Proof circuit MUST enforce the same per-note ownership
checks (note commitment integrity, Merkle path validity, nullifier
derivation, diversified address integrity, and nullifier
non-membership) as the Batched Claim circuit defined
in [^balance-proof]. This section specifies only the conditions that
extend beyond the Balance Proof.

#### Public Inputs

Given a primary input:

- $\mathsf{signed}\_{\mathsf{note}\_\mathsf{nullifier}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ —
  nullifier of the dummy signed note (for spend authorization binding).
- $\mathsf{rk} ⦂ \mathsf{SpendAuthSig}^{\mathsf{Orchard}}\mathsf{.Public}$ —
  randomized spend authorization verification key.
- $\mathsf{rt}^{\mathsf{cm}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — Ironwood pool
  note commitment tree root at the snapshot height.
- $\mathsf{rt}^{\mathsf{excl}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — nullifier
  non-membership tree root at the snapshot height.
- $\mathsf{van} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — the initial VAN
  commitment.
- $\mathsf{gov}\_{\mathsf{null}\_\mathsf{1}}, \ldots, \mathsf{gov}\_{\mathsf{null}\_\mathsf{5}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ —
  governance nullifiers for each note slot.
- $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
- $\mathsf{cmx}\_\mathsf{new} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — output note
  commitment to the governance hotkey address (PCZT scaffolding; not
  inserted into any on-chain tree).

#### Auxiliary Inputs

The prover knows, in addition to the per-note witnesses defined
in [^balance-proof]:

- $\mathsf{vpk} = (\mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}}, \mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}})$ —
  governance hotkey diversified address.
- $\mathsf{gov}\_{\mathsf{comm}\_\mathsf{rand}}$ — VAN commitment randomness.
- Signed note data: $\mathsf{g}\_\mathsf{d}^{\mathsf{signed}}, \mathsf{pk}\_\mathsf{d}^{\mathsf{signed}},
  \text{ρ}^{\mathsf{signed}}, \text{ψ}^{\mathsf{signed}},
  \mathsf{rcm}^{\mathsf{signed}}, \mathsf{cm}^{\mathsf{signed}}$ (value = 0).
- Output note data (PCZT scaffolding): $\mathsf{g}\_\mathsf{d}^{\mathsf{new}}, \mathsf{pk}\_\mathsf{d}^{\mathsf{new}},
  \mathsf{v}^{\mathsf{new}}, \text{ρ}^{\mathsf{new}},
  \text{ψ}^{\mathsf{new}}, \mathsf{rcm}^{\mathsf{new}}$ (address = hotkey).
  $\mathsf{v}^{\mathsf{new}}$ is not constrained by the circuit.

#### Conditions

**Per-note conditions (5 note slots, with padding).** For each note
$i \in \{1 \ldots 5\}$, the circuit MUST enforce the following
conditions from the Batched Claim circuit [^balance-proof]:

- Note commitment integrity.
- Merkle path validity in $\mathsf{rt}^{\mathsf{cm}}$ (skipped for padded
  notes).
- Diversified address integrity (same $\mathsf{ivk}$ MUST own all notes).
- Standard nullifier derivation (kept private).
- Nullifier non-membership in $\mathsf{rt}^{\mathsf{excl}}$ (skipped for padded
  notes).
- Padded notes MUST have value 0.

**Governance nullifier derivation.** For each real note $i$, the
circuit MUST enforce:

$$\mathsf{gov}\_{\mathsf{null}\_\mathsf{i}} = \mathsf{Poseidon}\bigl(\mathsf{nk}, \mathsf{tag}_{\mathsf{gov}}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{nf}^{\mathsf{old}}_i\bigr)$$

where $\mathsf{tag}_{\mathsf{gov}}$ is the field-element encoding of
"governance authorization" and $\mathsf{nf}^{\mathsf{old}}_i$ is the note's
standard nullifier (computed in-circuit but never revealed). This is an
instantiation of the alternate nullifier derivation defined
in [^balance-proof], with $\mathsf{tag} = \mathsf{tag}_{\mathsf{gov}}$ and
$\mathsf{dom} = \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$.

**Signed note integrity.** The signed note MUST be a dummy note with
value 0. The circuit MUST enforce that
$\mathsf{cm}^{\mathsf{signed}}$ is correctly constructed and that
$\mathsf{signed}\_{\mathsf{note}\_\mathsf{nullifier}}$ is correctly derived from it.

**Rho binding.** The circuit MUST enforce that
$\text{ρ}^{\mathsf{signed}}$ is deterministically bound to the
delegation context:

$$\text{ρ}^{\mathsf{signed}} = \mathsf{Poseidon}\bigl(\mathsf{cmx}\_\mathsf{1}, \mathsf{cmx}\_\mathsf{2}, \mathsf{cmx}\_\mathsf{3}, \mathsf{cmx}\_\mathsf{4}, \mathsf{cmx}\_\mathsf{5}, \mathsf{van}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}\bigr)$$

This makes the spend authorization signature non-replayable and scoped
to the exact delegation context.

**Spend authority.** The circuit MUST enforce that $\mathsf{rk} = \mathsf{SpendAuthSig}^{\mathsf{Orchard}}\mathsf{.RandomizePublic}(\alpha, \mathsf{ak}^{\mathbb{P}})$.

**Diversified address integrity for signed note.** The circuit MUST
enforce that the signed note's address belongs to
$(\mathsf{ak}, \mathsf{nk})$.

**Output note commitment.** The circuit MUST enforce that
$\mathsf{cmx}\_\mathsf{new}$ is a correctly constructed note commitment
to an output note addressed to the governance hotkey. The output note
exists solely to satisfy the PCZT signing flow: a standard Orchard
Action requires exactly one spend and one output, so the governance
PCZT includes a minimal output to the hotkey address.

$\mathsf{cmx}\_\mathsf{new}$ is not inserted into any commitment tree
and has no use beyond proof verification; it serves only to make the
hardware wallet's signed sighash commit to a complete Action structure.

See [^balance-proof] for the PCZT construction details and
[Why a Dummy Signed Note] for the design rationale.

**VAN integrity.** The circuit MUST enforce that the public VAN
commitment matches the claimed governance hotkey, ballot count, round,
and full proposal authority (using the two-layer construction defined
in [Vote Authority Note (VAN)]):

$$\mathsf{van}\_\mathsf{core} = \mathsf{Poseidon}\bigl(\mathsf{DOMAIN}\_\mathsf{VAN}, \mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}}, \mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}}, \mathsf{num}\_\mathsf{ballots}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{MAX}\_{\mathsf{PROPOSAL}\_\mathsf{AUTHORITY}}\bigr)$$

$$\mathsf{van} = \mathsf{Poseidon}\bigl(\mathsf{van}\_\mathsf{core}, \mathsf{gov}\_{\mathsf{comm}\_\mathsf{rand}}\bigr)$$

where $\mathsf{MAX}\_{\mathsf{PROPOSAL}\_\mathsf{AUTHORITY}} = 2^{16} - 1$.

**Ballot scaling.** The circuit MUST enforce that
$\mathsf{num}\_\mathsf{ballots} = \lfloor \sum v_i / 12{,}500{,}000 \rfloor$
with $\mathsf{num}\_\mathsf{ballots} \geq 1$, as defined in [Ballot Scaling].

#### Out-of-Circuit Verification

A delegation transaction (see [Delegation Transaction]) carrying a
Delegation Proof $\pi$ and a spend authorization signature $\sigma$ is
effective if and only if all of the following hold, evaluated against
the round's derived state as of the effective transactions recorded
before it (see [Effective Transactions]):

1. $\pi$ verifies against the public inputs.
2. $\mathsf{sighash}\_\mathsf{del}$ is exactly 32 bytes.
3. $\sigma$ is a valid $\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$
   signature on $\mathsf{sighash}\_\mathsf{del}$ under $\mathsf{rk}$.
4. $\mathsf{rt}^{\mathsf{cm}}$ equals the round's
   $\mathsf{nc}\_\mathsf{root}$ and $\mathsf{rt}^{\mathsf{excl}}$
   equals the round's $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$,
   as carried in its round creation transaction.
5. No $\mathsf{gov}\_{\mathsf{null}\_\mathsf{i}}$ appears in the round's
   governance nullifier set. If any does, the transaction is a
   double-delegation.
6. $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ names a round in whose
   voting window the transaction is recorded (see [Round Lifecycle]).

The effects of an effective delegation transaction are:

7. $\mathsf{van}$ is inserted into the round's VCT.
8. Every $\mathsf{gov}\_{\mathsf{null}\_\mathsf{i}}$ is added to the
   governance nullifier set.

### Delegation Sighash

The delegation sighash $\mathsf{sighash}\_\mathsf{del}$ is a
client-provided 32-byte value included in the delegation message. For
hardware wallet flows, the client computes it as the ZIP 244 [^zip-244]
shielded transaction sighash of a governance PCZT [^pczt]; the
construction of this PCZT is specified
in [^balance-proof]. For software wallets, the client signs the
sighash directly without PCZT construction.

A verifier checks the
$\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$ against this
client-provided sighash without recomputing it. The sighash does not
need to be independently reconstructible by a verifier because the
Delegation Proof provides the governance data binding: the ZKP proves
that $\mathsf{rk}$ is a valid rerandomization of the holder's spend
authorization key, that the holder owns the claimed notes, and that the
VAN commitment is correctly constructed. An attacker who substitutes a
different sighash cannot produce a valid signature under
$\mathsf{rk}$ without knowledge of the holder's spending key.

## Vote Phase

### Vote Proof

The Vote Proof demonstrates that a holder of a valid VAN is casting a
vote: consuming the old VAN, producing a new VAN with decremented
proposal authority, and constructing a Vote Commitment that binds $N_s$
El Gamal-encrypted shares to the chosen proposal and decision. The
Vote Proof circuit MUST enforce all conditions specified below.

#### Public Inputs

Given a primary input:

- $\mathsf{van}\_\mathsf{nullifier} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ —
  nullifier of the old VAN (prevents double-voting).
- $\mathsf{r}\_\mathsf{vpk} ⦂ \mathsf{SpendAuthSig}^{\mathsf{Orchard}}\mathsf{.Public}$ —
  randomized voting public key.
- $\mathsf{van}\_{\mathsf{new}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — the new VAN
  commitment with decremented proposal authority.
- $\mathsf{vc} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — the vote commitment.
- $\mathsf{rt}^{\mathsf{vct}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — root of the
  Vote Commitment Tree.
- $\mathsf{anchor}\_\mathsf{height} ⦂ \mathbb{N}$ — the vote chain
  height whose VCT root is the anchor.
- $\mathsf{proposal}\_\mathsf{id} ⦂ \{1 \ldots \mathsf{MAX}\_\mathsf{PROPOSALS}\}$ —
  which proposal.
- $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
- $\mathsf{ea}\_\mathsf{pk} ⦂ \mathbb{P}^*$ — election authority public key
  (x and y coordinates).

#### Auxiliary Inputs

The prover knows:

- $\mathsf{vote}\_\mathsf{decision} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — the voter's choice.
- $\mathsf{vsk.ak} ⦂ \mathbb{P}^*$ — voting spend authorization
  validating key.
- $\mathsf{vsk.nk} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — nullifier
  deriving key from the governance hotkey's full viewing key.
- $\mathsf{rivk}\_\mathsf{v} ⦂ \mathsf{Commit}^{\mathsf{ivk}}\mathsf{.Trapdoor}$ —
  CommitIvk randomness for the voting key.
- $\alpha_v$ — spend authorization randomizer for the voting hotkey.
- $\mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}} ⦂ \mathbb{P}^*$ — diversified base point from the
  VAN (full point; $\mathsf{Extract}\_{\mathbb{P}}$ applied for VAN integrity hash).
- $\mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}} ⦂ \mathbb{P}^*$ — diversified transmission
  key point from the VAN (full point; $\mathsf{Extract}\_{\mathbb{P}}$ applied for
  VAN integrity hash).
- $\mathsf{num}\_\mathsf{ballots}$ — total voting weight in ballots.
- $\mathsf{proposal}\_{\mathsf{authority}\_\mathsf{old}}$,
  $\mathsf{proposal}\_{\mathsf{authority}\_\mathsf{new}}$ — old and new bitmasks.
- $\mathsf{gov}\_{\mathsf{comm}\_\mathsf{rand}}$ — VAN commitment randomness (shared
  between old and new VAN).
- $\mathsf{path}^{\mathsf{vct}}, \mathsf{pos}^{\mathsf{vct}}$ — Merkle proof for the
  old VAN in the VCT.
- $\mathsf{van}_{\mathsf{old}}$ — old VAN commitment.
- $\mathsf{v}\_\mathsf{0}, \ldots, \mathsf{v}_{N_s - 1}$ — plaintext share values.
- $r_0, \ldots, r_{N_s - 1}$ — El Gamal encryption randomness per share.
- $\mathsf{blind}\_\mathsf{0}, \ldots, \mathsf{blind}_{N_s - 1}$ — per-share blind
  factors.

#### Conditions

##### VAN Ownership and Spending

**Condition 1: Merkle tree membership.** The circuit MUST enforce that
the old VAN exists in the VCT:
$(\mathsf{path}^{\mathsf{vct}}, \mathsf{pos}^{\mathsf{vct}})$ MUST be
a valid Merkle path of depth $\mathsf{MerkleDepth}^{\mathsf{vct}}$
from $\mathsf{van}_{\mathsf{old}}$ to the anchor
$\mathsf{rt}^{\mathsf{vct}}$, using Poseidon for internal node hashing.

**Condition 2: Old VAN integrity.** The circuit MUST enforce that
the old VAN commitment matches the claimed fields (using the two-layer
construction defined in [Vote Authority Note (VAN)]). The
$\mathsf{vpk}$ values in the Poseidon input are x-coordinates, i.e.,
$\mathsf{Extract}\_{\mathbb{P}}$ of the corresponding full points:

$$\mathsf{van}\_{\mathsf{core}\_\mathsf{old}} = \mathsf{Poseidon}\bigl(\mathsf{DOMAIN}\_\mathsf{VAN}, \mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}}, \mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}}, \mathsf{num}\_\mathsf{ballots}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{proposal}\_{\mathsf{authority}\_\mathsf{old}}\bigr)$$

$$\mathsf{van}\_\mathsf{old} = \mathsf{Poseidon}\bigl(\mathsf{van}\_{\mathsf{core}\_\mathsf{old}}, \mathsf{gov}\_{\mathsf{comm}\_\mathsf{rand}}\bigr)$$

**Condition 3: Diversified address integrity.** The circuit MUST
enforce that the VAN's address belongs to the voting key:

$$\mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}} = [\mathsf{ivk}\_\mathsf{v}]\, \mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}}$$

where $\mathsf{ivk}\_\mathsf{v} = \mathsf{CommitIvk}_{\mathsf{rivk}\_\mathsf{v}}\!\bigl(\mathsf{Extract}_{\mathbb{P}}(\mathsf{vsk.ak}),\; \mathsf{vsk.nk}\bigr)$ [^protocol-concretecommitivk].

**Condition 4: Spend authority.** The circuit MUST enforce that the
randomized voting public key is a valid rerandomization:

$$\mathsf{r}\_\mathsf{vpk} = \mathsf{vsk.ak} + [\alpha_v]\, G$$

**Condition 5: VAN nullifier.** The circuit MUST enforce that the
public $\mathsf{van}\_\mathsf{nullifier}$ is correctly derived:

$$\mathsf{van}\_\mathsf{nullifier} = \mathsf{Poseidon}\bigl(\mathsf{vsk.nk}, \mathsf{tag}_{\mathsf{van}}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{van}_{\mathsf{old}}\bigr)$$

where $\mathsf{tag}_{\mathsf{van}}$ is the field-element encoding of
`"vote authority spend"`.

See [Why VAN Nullifier Domain Separation].

##### New VAN Construction

**Condition 6: Proposal authority decrement.** The circuit MUST
enforce that bit $\mathsf{proposal}\_\mathsf{id}$ is cleared in the
authority bitmask:

- $\mathsf{proposal}\_{\mathsf{authority}\_\mathsf{old}}$ MUST be decomposed into
  51 boolean wires $b_0, \ldots, b_{50}$ that recompose to the original
  value.
- The bit at position $\mathsf{proposal}\_\mathsf{id}$ MUST be 1 (the voter
  has authority for this proposal).
- $\mathsf{proposal}\_{\mathsf{authority}\_\mathsf{new}}$ MUST be the recomposition
  with bit $\mathsf{proposal}\_\mathsf{id}$ cleared; all other bits MUST be
  unchanged.

**Condition 7: New VAN integrity.** The circuit MUST enforce that the
new VAN is correctly constructed (using the two-layer construction
defined in [Vote Authority Note (VAN)]):

$$\mathsf{van}\_{\mathsf{core}\_\mathsf{new}} = \mathsf{Poseidon}\bigl(\mathsf{DOMAIN}\_\mathsf{VAN}, \mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}}, \mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}}, \mathsf{num}\_\mathsf{ballots}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{proposal}\_{\mathsf{authority}\_\mathsf{new}}\bigr)$$

$$\mathsf{van}\_\mathsf{new} = \mathsf{Poseidon}\bigl(\mathsf{van}\_{\mathsf{core}\_\mathsf{new}}, \mathsf{gov}\_{\mathsf{comm}\_\mathsf{rand}}\bigr)$$

The new VAN MUST reuse the old VAN's diversified address and commitment
randomness; only $\mathsf{proposal}\_\mathsf{authority}$ changes.

##### Vote Commitment Construction

**Condition 8: Shares sum correctness.** The circuit MUST enforce:

$$\sum_{i=0}^{N_s - 1} \mathsf{v}\_\mathsf{i} = \mathsf{num}\_\mathsf{ballots}$$

**Condition 9: Shares range check.** The circuit MUST enforce that
each share is bounded:

$$0 \leq \mathsf{v}\_\mathsf{i} < 2^{30} \quad \text{for each } i \in \{0 \ldots N_s - 1\}$$

This bound is critical for two reasons: (1) it ensures the base-field
share sum and the scalar-field El Gamal encoding agree (no modular
reduction in either field), and (2) it keeps the aggregate discrete log
small enough for efficient recovery at tally time (see [Decryption]).

**Condition 10: Shares hash integrity.** The circuit MUST enforce that
the blinded share commitments and their aggregate hash are correctly
computed:

$$\mathsf{share}\_{\mathsf{comm}\_\mathsf{i}} = \mathsf{Poseidon}\bigl(\mathsf{blind}\_\mathsf{i}, C_{1,i,x}, C_{2,i,x}, C_{1,i,y}, C_{2,i,y}\bigr) \quad \text{for each } i$$

$$\mathsf{shares}\_\mathsf{hash} = \mathsf{Poseidon}\bigl(\mathsf{share}\_{\mathsf{comm}\_\mathsf{0}}, \ldots, \mathsf{share}\_{\mathsf{comm}_{N_s - 1}}\bigr)$$

**Condition 11: El Gamal encryption integrity.** The circuit MUST
enforce that each ciphertext is a valid encryption of its share under
$\mathsf{ea}\_\mathsf{pk}$:

$$C_{1,i} = [r_i]\, G$$
$$C_{2,i} = [\mathsf{v}\_\mathsf{i}]\, G + [r_i]\, \mathsf{ea}\_\mathsf{pk}$$

The circuit MUST constrain equality on both the $x$- and
$y$-coordinates of the computed and witnessed ciphertext points. The
$y$-coordinates are needed because they appear in the share commitment
(condition 10); the ECC gadget already produces full curve points, so
no additional decomposition cost is incurred.
See [Why Share Commitments Bind Full Curve Points].

**Condition 12: Vote commitment integrity.** The circuit MUST enforce
that the public vote commitment matches the private vote details:

$$\mathsf{vc} = \mathsf{Poseidon}\bigl(\mathsf{DOMAIN}\_\mathsf{VC}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{shares}\_\mathsf{hash}, \mathsf{proposal}\_\mathsf{id}, \mathsf{vote}\_\mathsf{decision}\bigr)$$

#### Out-of-Circuit Verification

A vote transaction (see [Vote Transaction]) carrying a Vote Proof $\pi$
and a vote spend authorization signature $\sigma$ is effective if and
only if all of the following hold, evaluated against the round's
derived state as of the effective transactions recorded before it:

1. $\pi$ verifies against the public inputs.
2. $\sigma$ is a valid $\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$
   signature on the vote sighash computed from the transaction's
   fields (see [Vote Sighash]), under $\mathsf{r}\_\mathsf{vpk}$.
3. $\mathsf{van}\_\mathsf{nullifier}$ does not appear in the round's VAN
   nullifier set. If it does, the transaction is a double-vote.
4. $\mathsf{anchor}\_\mathsf{height}$ is at least the round's creation
   height and less than the height at which the transaction is
   recorded, and $\mathsf{rt}^{\mathsf{vct}}$ equals the root of the
   round's VCT after the last effective transaction recorded at or
   before $\mathsf{anchor}\_\mathsf{height}$.
5. $\mathsf{proposal}\_\mathsf{id}$ is the identifier of one of the
   round's proposals.
6. $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ names a round in whose
   voting window the transaction is recorded (see [Round Lifecycle]).
7. $\mathsf{ea}\_\mathsf{pk}$ equals the round's election authority
   public key as derived from its ceremony (see
   [Election Authority Key Ceremony]).

The effects of an effective vote transaction are:

8. $\mathsf{van}_{\mathsf{new}}$ and then $\mathsf{vc}$ are inserted
   into the round's VCT.
9. $\mathsf{van}\_\mathsf{nullifier}$ is added to the VAN nullifier set.

Note: $\mathsf{vote}\_\mathsf{decision}$ is a private witness in the
Vote Proof and is never published. Its range is enforced when the share
is revealed, by condition 6 of the [Vote Reveal Proof]; a decision at
or beyond the proposal's option count is aggregated into a position
the tally never decrypts.

### Vote Sighash

The vote sighash is computed by any verifier from the vote
transaction's fields. Unlike the delegation sighash, it is not
client-provided; the governance hotkey is software-controlled and signs
the same digest any verifier computes.

The vote sighash is $\mathsf{BLAKE2b\text{-}256}(\mathsf{vote}\_\mathsf{payload})$
using an unkeyed BLAKE2b-256 hash (no personalization parameter).
$\mathsf{vote}\_\mathsf{payload}$ is the concatenation of a domain
prefix followed by the message fields, each zero-padded to 32 bytes:

| Component | Encoding | Size |
|---|---|---|
| `"SVOTE_CAST_VOTE_SIGHASH_V0"` | raw ASCII bytes | 26 |
| $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ | field element, zero-padded to 32 | 32 |
| $\mathsf{r}\_\mathsf{vpk}$ | compressed point, zero-padded to 32 | 32 |
| $\mathsf{van}\_\mathsf{nullifier}$ | field element, zero-padded to 32 | 32 |
| $\mathsf{van}\_{\mathsf{new}}$ | field element, zero-padded to 32 | 32 |
| $\mathsf{vc}$ | field element, zero-padded to 32 | 32 |
| $\mathsf{proposal}\_\mathsf{id}$ | 4-byte LE integer, zero-padded to 32 | 32 |
| $\mathsf{anchor}\_\mathsf{height}$ | 8-byte LE integer, zero-padded to 32 | 32 |

Total payload: 250 bytes. Pallas field elements and compressed points
use their canonical little-endian
encoding [^protocol-pallasandvesta], right-padded with zero bytes to
fill 32 bytes when shorter. Integer fields are encoded as unsigned
little-endian and right-padded with zero bytes to 32 bytes.

The voter's client MUST compute the same digest and sign it with
the governance hotkey's spend-authorizing key (rerandomized by
$\alpha_v$) before submitting the vote transaction.

## Share Reveal Phase

Share reveal takes place during the reveal window, after the voting
window has closed and the round's VCT has been frozen (see
[Round Lifecycle]). The voter's client constructs one Vote Reveal Proof
per share and submits the resulting messages as specified in
[Share Submission]. No party other than the voter's client constructs a
Vote Reveal Proof or handles a finished message before it is recorded;
see [Why the Client Constructs Reveal Proofs] and
[Why There Are No Relays].

### Vote Reveal Proof

The Vote Reveal Proof opens a single encrypted share from a Vote
Commitment for homomorphic accumulation, without revealing the
plaintext amount, which Vote Commitment the share came from, or which
option the share supports. It publishes a vector of
$N_{\mathsf{opt}} = 8$ El Gamal ciphertexts, one per option position:
the position matching the vote's decision carries the share's
committed ciphertext, and every other position carries a fresh
encryption of zero. The Vote Reveal Proof circuit MUST enforce all
conditions specified below.

#### Public Inputs

Given a primary input:

- $\mathsf{share}\_\mathsf{nullifier} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ —
  prevents double-counting.
- $E_{j,1,x}, E_{j,1,y}, E_{j,2,x}, E_{j,2,y} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
  for each $j \in \{0 \ldots N_{\mathsf{opt}} - 1\}$ — the coordinates
  of the option-vector ciphertexts $E_j = (E_{j,1}, E_{j,2})$.
- $\mathsf{proposal}\_\mathsf{id} ⦂ \{1 \ldots \mathsf{MAX}\_\mathsf{PROPOSALS}\}$ —
  which proposal.
- $\mathsf{rt}^{\mathsf{vct}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — the
  round's final VCT root.
- $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
- $\mathsf{ea}\_\mathsf{pk} ⦂ \mathbb{P}^*$ — election authority public
  key ($x$ and $y$ coordinates).

#### Auxiliary Inputs

The prover (the voter's client) knows:

- $\mathsf{vc} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$ — the vote commitment
  being opened (hidden from the verifier).
- $\mathsf{path}^{\mathsf{vct}}, \mathsf{pos}^{\mathsf{vct}}$ — Merkle proof for the VC
  in the VCT.
- $\mathsf{shares}\_\mathsf{hash} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
- $\mathsf{vote}\_\mathsf{decision} ⦂ \{ 0 \ldots N_{\mathsf{opt}} - 1 \}$
  — the voter's choice.
- $\mathsf{share}\_\mathsf{index} \in \{0, 1, \ldots, N_s - 1\}$ — which share is being revealed.
- $\mathsf{share}\_{\mathsf{comm}\_0} \ldots \mathsf{share}\_{\mathsf{comm}\_{N_s - 1}}$ —
  all $N_s$ blinded share commitments (to recompute
  $\mathsf{shares}\_\mathsf{hash}$).
- $\mathsf{blind}$ — the blind factor for the revealed share
  (at position $\mathsf{share}\_\mathsf{index}$).
- $C_1, C_2 ⦂ \mathbb{P}^*$ — the committed ciphertext of the revealed
  share, $\mathsf{enc}\_{\mathsf{share}\_{\mathsf{share}\_\mathsf{index}}}$.
- $\rho_0, \ldots, \rho_{N_{\mathsf{opt}} - 1}$ — padding randomness,
  one Pallas scalar per option position, sampled uniformly at random.

#### Conditions

##### Vote Commitment Membership

**Condition 1: Merkle tree membership.** The circuit MUST enforce that
the VC exists in the VCT:
$(\mathsf{path}^{\mathsf{vct}}, \mathsf{pos}^{\mathsf{vct}})$ MUST be
a valid Merkle path from $\mathsf{vc}$ to $\mathsf{rt}^{\mathsf{vct}}$,
without revealing which leaf. The VC value is a private witness.

**Condition 2: Vote commitment integrity.** The circuit MUST enforce
that the VC is correctly constructed from its components:

$$\mathsf{vc} = \mathsf{Poseidon}\bigl(\mathsf{DOMAIN}\_\mathsf{VC}, \mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}, \mathsf{shares}\_\mathsf{hash}, \mathsf{proposal}\_\mathsf{id}, \mathsf{vote}\_\mathsf{decision}\bigr)$$

This binds the public $\mathsf{proposal}\_\mathsf{id}$ and the private
$\mathsf{vote}\_\mathsf{decision}$ to the private VC, ensuring that the
revealed share is attributed to the correct proposal and, through
condition 7, to the option the voter committed to.

##### Share Opening

**Condition 3: Shares hash integrity.** The circuit MUST enforce that
$\mathsf{shares}\_\mathsf{hash}$ is recomputed from the witness share
commitments:

$$\mathsf{shares}\_\mathsf{hash} = \mathsf{Poseidon}\bigl(\mathsf{share}\_{\mathsf{comm}\_\mathsf{0}}, \ldots, \mathsf{share}\_{\mathsf{comm}_{N_s - 1}}\bigr)$$

The recomputed $\mathsf{shares}\_\mathsf{hash}$ MUST equal the one inside
the VC (via condition 2). The share commitments are blinded
(see [Why Blinded Share Commitments]), so a chain observer cannot
recompute them from revealed ciphertexts.

**Condition 4: Share membership.** The circuit MUST enforce that the
commitment derived from the witness ciphertext coordinates
$C_{1,x}, C_{2,x}, C_{1,y}, C_{2,y}$ and the witness blind factor
matches the share commitment at position
$\mathsf{share}\_\mathsf{index}$:

$$\mathsf{Poseidon}\bigl(\mathsf{blind}_{\mathsf{share}\_\mathsf{index}}, C_{1,x}, C_{2,x}, C_{1,y}, C_{2,y}\bigr) = \mathsf{share}\_\mathsf{comms}[\mathsf{share}\_\mathsf{index}]$$

The circuit MUST encode $\mathsf{share}\_\mathsf{index}$ as a one-hot
selector vector over $N_s$ positions. The mux MUST extract the
corresponding $\mathsf{share}\_\mathsf{comm}$ via a dot product and
constrain equality with the commitment derived from the witness
ciphertext coordinates and the witness blind factor. Only the blind
factor for the revealed share is needed; the remaining $N_s - 1$ share
commitments used in condition 3 are opaque witnesses.

##### Nullifier

**Condition 5: Share nullifier.** The circuit MUST enforce that the
public $\mathsf{share}\_\mathsf{nullifier}$ is correctly derived:

$$\mathsf{share}\_\mathsf{nullifier} = \mathsf{Poseidon}\bigl(\mathsf{tag}_{\mathsf{share}}, \mathsf{vc}, \mathsf{share}\_\mathsf{index}, \mathsf{blind}\bigr)$$

where $\mathsf{tag}_{\mathsf{share}}$ is the field-element encoding of
`"share spend"` and $\mathsf{blind}$ is the blind factor for the
revealed share. The VC and blind are private, making the nullifier
unlinkable to a specific VC without knowledge of these witnesses.

##### Option Vector

**Condition 6: Decision selector.** The circuit MUST derive a one-hot
selector $s_0, \ldots, s_{N_{\mathsf{opt}} - 1}$ from the private
decision and enforce:

$$s_j \in \{0, 1\} \text{ for each } j, \qquad \sum_{j} s_j = 1, \qquad \sum_{j} j \cdot s_j = \mathsf{vote}\_\mathsf{decision}$$

These constraints also bound $\mathsf{vote}\_\mathsf{decision}$ to
$\{0 \ldots N_{\mathsf{opt}} - 1\}$; a VC whose decision lies outside
that range cannot be opened.

**Condition 7: Option vector integrity.** For each position
$j \in \{0 \ldots N_{\mathsf{opt}} - 1\}$, the circuit MUST compute a
padding ciphertext

$$P_j = \bigl([\rho_j]\, G,\ [\rho_j]\, \mathsf{ea}\_\mathsf{pk}\bigr)$$

which is an encryption of zero under $\mathsf{ea}\_\mathsf{pk}$ per
[Encryption], and enforce coordinate-wise that

$$E_j = s_j \cdot (C_1, C_2) + (1 - s_j) \cdot P_j$$

That is, the public ciphertext at the decision position equals the
committed share ciphertext, and the public ciphertext at every other
position is a fresh encryption of zero. The circuit MUST compute $P_j$
for every position, including the decision position where it is
discarded, so that the circuit's structure does not depend on the
decision. The circuit MUST constrain equality on both coordinates of
each point.

Each $\rho_j$ MUST be sampled independently. Reusing one scalar across
positions would let an observer subtract two positions' second
components and recover the share value by a bounded discrete logarithm;
see [Why Decisions Are Encrypted at Reveal].

#### Out-of-Circuit Verification

A share reveal transaction (see [Share Reveal Transaction]) carrying a
Vote Reveal Proof $\pi$ is effective if and only if all of the
following hold, evaluated against the round's derived state as of the
effective transactions recorded before it:

1. $\pi$ verifies against the public inputs.
2. $\mathsf{share}\_\mathsf{nullifier}$ does not appear in the round's
   share nullifier set. If it does, the transaction is a
   double-counting.
3. $\mathsf{rt}^{\mathsf{vct}}$ equals the round's final VCT root (see
   [Round Lifecycle]).
4. $\mathsf{proposal}\_\mathsf{id}$ is the identifier of one of the
   round's proposals.
5. $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ names a round in whose
   reveal window the transaction is recorded.
6. $\mathsf{ea}\_\mathsf{pk}$ equals the round's election authority
   public key.

The effect of an effective share reveal transaction is:

7. $\mathsf{share}\_\mathsf{nullifier}$ is added to the share nullifier
   set.

The option-vector ciphertexts $E_0, \ldots, E_{N_{\mathsf{opt}} - 1}$
of every effective share reveal are aggregated per
$(\mathsf{proposal}\_\mathsf{id}, j)$ pair by whoever tallies or
verifies the round, after the reveal window closes; see [Aggregation].
Nothing is aggregated as transactions are recorded.

A verifier does not learn, and does not check, which position carries
the share's value. A share whose committed decision is at or beyond the
proposal's option count is aggregated into a position that [Tally]
never decrypts; it contributes to no option, and nothing in this
protocol allows a misplaced share to alter another option's total.

### Share Submission

A share reveal message reaches the vote chain from the voter's client
and from no other party. The client constructs every Vote Reveal Proof,
assembles every message, and submits each one itself, at a height it
draws, over a network path used for nothing else.

**Constructing the messages.** Once the round enters REVEALING, the
client MUST obtain the round's final VCT root and a Merkle path for its
VC against that root, construct the Vote Reveal Proof for each share
$i \in \{0 \ldots N_s - 1\}$ per [Vote Reveal Proof], and assemble the
$N_s$ share reveal transactions (see [Share Reveal Transaction]). A
client MUST NOT send any of the auxiliary inputs of the Vote Reveal
Proof — the vote commitment, its VCT position or path, the shares hash,
the share commitments, the blind factors or the committed ciphertexts —
to any other party, and MUST NOT hand a finished message to any party
other than a vote chain node, for immediate submission, at the
message's drawn height. The client SHOULD construct and persist all
$N_s$ messages at once, when it draws its schedule, so that each later
submission needs no further computation or synchronisation (see
[Submission Timing]).

**Independence of submissions.** The $N_s$ messages of one vote MUST be
submitted as if by $N_s$ unrelated clients:

1. Each message MUST be submitted over a network connection that shares
   no identifying state with the connection used for any other message
   of the same vote — for example, a separate Tor circuit or mixnet
   channel per message. A client MUST NOT reuse a circuit, a source
   address it controls the visibility of, or a session across two
   messages of one vote.
2. Each message MUST be submitted at the height drawn for it as
   specified in [Submission Timing], or under the catch-up rules
   there.
3. A client MAY submit each message to any vote chain node, including
   one it operates, and MAY use a different node per message. A node
   forwards what it receives to the rest of the network with its own
   network identity as the origin, so a client that submits through a
   node it operates MUST apply rule 1 to that node's connections as
   well, or that node becomes the common origin of every message.

**Being online.** Direct submission requires the client to be online at
each drawn height. No party in this protocol accepts a message for
later submission, and a client MUST NOT delegate submission to any
party (see [Why There Are No Relays]). A client that is not running at
a drawn height submits under the catch-up rules in
[Submission Timing].

**Retry.** A client SHOULD confirm that each of its messages has been
recorded on the vote chain. A client that observes that a message has
not been recorded within a client-configured number of blocks MAY
resubmit it, to a different node, under rule 1 above. Because a share
nullifier is effective once, a duplicate is not effective and is
harmless. Confirming inclusion by querying a node for the client's own
share nullifiers reveals which nullifiers are the client's; see
[Open issues].

### Submission Timing

Share submission follows the scheduling discipline that ZIP 318
[^zip-0318] specifies for pool-crossing transfers. The two problems are
the same: a client emits several transactions that together reveal a
quantity it wishes to keep private, and an observer who can group them
recovers that quantity. ZIP 318 addresses it with randomized
decomposition, randomized ordering, memoryless inter-arrival delays,
and rules for a wallet that is not running when a transaction falls
due. [Vote Share] already supplies the first. This section supplies
the rest.

Heights below are vote chain block heights, which is what an observer
of the record measures. A client converts between heights and time
using the vote chain's published block rate (see [Deployment]).

Let $H_{\mathsf{end}}$ be the round's
$\mathsf{reveal}\_\mathsf{end}\_\mathsf{height}$, let $H_0$ be the
height at which the client commits a schedule, which is no earlier
than the start of the reveal window, and let
$W = H_{\mathsf{end}} - H_0 - \Delta$, where $\Delta$ is a
deployment-specified safety margin, in blocks, covering inclusion (see
[Deployment]).

A client constructing a submission schedule:

1. MUST shuffle the $N_s$ shares into a uniformly random order before
   assigning submission heights, so that the sequence of share values a
   client emits is not a function of their magnitudes or of their
   indices within the vote commitment.
2. MUST assign submission heights by advancing a running height from
   $H_0$, drawing each successive delay independently from an
   exponential distribution with rate
   $\lambda = 1 / \mathsf{MEAN}\_\mathsf{DELAY}$, where
   $\mathsf{MEAN}\_\mathsf{DELAY} = W / (N_s + 1)$, rounded to a whole
   number of blocks.
3. MUST discard and redraw any delay exceeding
   $\mathsf{MAX}\_\mathsf{DELAY}$ (see [Deployment]).
4. MUST NOT impose a minimum separation between consecutive draws.
   Enforcing one would destroy the memorylessness that makes the
   schedule uninformative; occasional short gaps are a property of the
   distribution, not a defect.
5. MUST draw all randomness used in the shuffle and the delays from a
   cryptographically secure random number generator.

A message is due when the vote chain reaches its drawn height. The
client submits it then, over its own network path per
[Share Submission].

**Background submission.** Where the platform grants the wallet
background execution, the wallet SHOULD submit each message from a
background session at its drawn height. Background scheduling is
best-effort: the platform chooses the exact execution time within a
requested window, and a slip of a few blocks is normal operation. A
background session that submits a share reveal message MUST NOT also
synchronise the wallet's Zcash state or perform any other request that
identifies the wallet, so that the submission cannot be correlated with
that activity by the servers involved.

**Catching up.** On every application open during the reveal window,
the wallet MUST reconcile its schedule against the messages the vote
chain has recorded, and MUST surface any message that is overdue. The
wallet MUST NOT submit more than one overdue message in that session:
submitting several messages of one vote in close succession from one
session reproduces, through timing and origin, the grouping that the
schedule exists to prevent. The remaining overdue messages MUST be
rescheduled by redrawing their delays, under the rules above, from the
current height, and the wallet SHOULD tell the voter that the vote is
not yet fully revealed and when to return. On-open reconciliation is
the primary catch-up mechanism and MUST NOT rely on notification
delivery.

**When the window is short.** If the schedule, or a rescheduling under
the catch-up rules, would place any share after
$H_{\mathsf{end}} - \Delta$, the client MUST compress the schedule by
drawing each remaining share's submission height independently and
uniformly from the interval $[\mathsf{now}, H_{\mathsf{end}} - \Delta]$.
A client MUST NOT submit the remaining shares as a batch,
simultaneously, or in share-index order, and MUST NOT place more than
one share into a single vote chain block where it can observe block
boundaries. Where the remaining window is too short for the client to
submit all $N_s$ shares at all, the client MUST inform the voter that
the round is closing and that proceeding will submit shares in close
succession, rather than proceeding silently.

Submitting promptly is not a substitute for submitting independently. A
client that reacts to a closing window by sending everything at once
reproduces, through timing, the exposure that
[Why There Is No Single-Share Mode] removes from the payload.

There is no single-share submission mode. A client MUST NOT place a
voter's entire ballot count into one share, however close the deadline:
it concentrates the voter's entire weight into one ciphertext, so a
single decryption recovers it exactly. See
[Why There Is No Single-Share Mode].

## Tally

After the reveal window closes, any party can compute the round's tally
from the record: it aggregates the ciphertexts of the round's effective
share reveals, and combines the partial decryptions the trustees have
recorded. Each per-$(\mathsf{proposal}\_\mathsf{id}, j)$ aggregate
ciphertext, for each option position $j$ below the proposal's option
count, is decrypted by a threshold procedure. No party reconstructs
$\mathsf{ea}\_\mathsf{sk}$ at any point, and no step of the procedure
is performed by the vote chain.

Let $t$ be the round's decryption threshold, and let each of the
round's $n$ trustees $i \in \{1 \ldots n\}$ hold the key share
$\mathsf{sk}_i$ of $\mathsf{ea}\_\mathsf{sk}$, with trustee share key
$\mathsf{pk}_i = [\mathsf{sk}_i]\, G$, as produced by
[Election Authority Key Ceremony]. The shares are Shamir shares
[^shamir] on a polynomial of degree $t - 1$, so any $t$ of them suffice.

### Aggregation

For each $(\mathsf{proposal}\_\mathsf{id}, j)$ pair, the aggregate
ciphertext $(C_{1,\mathsf{agg}}, C_{2,\mathsf{agg}})$ is the
component-wise sum, per [Additive Homomorphism], of the position-$j$
ciphertext $E_j$ of every effective share reveal transaction for that
proposal (see [Vote Reveal Proof]). The sum does not depend on the
order of the terms. Any party holding the record computes it; it is
not maintained by the vote chain and no transaction carries it.

Positions at or beyond the proposal's option count aggregate the
padding ciphertexts of every reveal and the committed ciphertext of any
share whose decision was out of range. They MUST NOT be decrypted and
are not part of the tally.

### Partial Decryption

Each trustee $i$ that takes part in the tally computes the aggregate
ciphertexts itself from the record, as above, and records a partial
decryption transaction (see [Partial Decryption Transaction]) carrying,
for each $(\mathsf{proposal}\_\mathsf{id}, j)$ pair with $j$ below the
proposal's option count, in ascending order of proposal identifier and
then position,

$$D_i = [\mathsf{sk}_i]\, C_{1,\mathsf{agg}}$$

together with a Chaum-Pedersen DLEQ proof, as specified in
[Chaum-Pedersen DLEQ Proofs], instantiated with $P = \mathsf{pk}_i$,
$H = C_{1,\mathsf{agg}}$, $Q = D_i$ and witness $x = \mathsf{sk}_i$.
The proof demonstrates
$\log_G(\mathsf{pk}_i) = \log_{C_{1,\mathsf{agg}}}(D_i)$, establishing
that the share behind the trustee's share key is the share used to
compute $D_i$.

A partial decryption transaction is effective if and only if it is
recorded after the round's reveal end height, it is signed by the
trustee account key of a trustee of the round, no earlier effective
partial decryption transaction from the same trustee exists for the
round, and every DLEQ proof in it verifies against the verifier's own
$C_{1,\mathsf{agg}}$ for that pair. The verifier supplies
$C_{1,\mathsf{agg}}$ from its own aggregation, so a trustee that
aggregated a different set of reveals produces proofs that do not
verify, and its transaction is not effective. Without this check a
single trustee could publish a bogus $D_i$, and the resulting
combination would yield a point whose discrete logarithm search fails
or returns an unrelated value, with no indication of which trustee was
responsible.

### Combination

A round has a tally once effective partial decryption transactions
from at least $t$ distinct trustees exist. Given the verified partial
decryptions $\{(i, D_i)\}$ for a pair, from a set $S$ of trustees with
$|S| \ge t$, the Lagrange coefficients at $0$ are

$$\lambda_i = \prod_{j \in S,\, j \neq i} \frac{-j}{i - j}$$

and the combination in the exponent recovers

$$[\mathsf{ea}\_\mathsf{sk}]\, C_{1,\mathsf{agg}} = \sum_{i \in S} [\lambda_i]\, D_i$$

from which the aggregate plaintext follows by [Decryption]:

$$[\mathsf{total}\_\mathsf{value}]\, G = C_{2,\mathsf{agg}} - [\mathsf{ea}\_\mathsf{sk}]\, C_{1,\mathsf{agg}}$$

$\mathsf{total}\_\mathsf{value}$ is then recovered by baby-step giant-step,
and is the total for option $j$ of that proposal, in the units
specified in [Tally units]. Any $t$ of the effective partial
decryptions give the same result. A party that publishes a result
SHOULD state the set $S$ it used; [Verification] states what checking
a published result establishes.

Only the aggregate is decrypted. No step of this procedure reveals an
individual vote amount — but see [Privacy Implications] for what a
coalition holding $t$ shares can do outside this procedure, and
[Verification] for what a published tally does and does not establish.

## Voting Round

A voting round is the unit within which delegation, voting, share
reveal and tally take place. This section specifies what a round is —
its proposals and its Zcash snapshot — and the rules by which a round
is created, opened, and closed: the lifecycle, creation, the poll
runner's signature, the election authority key ceremony that produces
the round's key, the ratification that opens voting, and cancellation.
Every rule here is a rule for interpreting the vote chain's record,
applied by any party that reads it (see [Vote Chain Record]); the vote
chain enforces none of them. Who performs each step, and how operators
are organised to do so, is specified in
`draft-valargroup-shielded-voting-setup` [^voting-setup].

### Round Lifecycle

A round's state at any vote chain height is a function of the record
up to that height. Let $H_c$ be the height at which the round's
effective round creation transaction is recorded.

1. **PENDING**: from $H_c$, awaiting the election authority key
   ceremony and its ratification by every trustee (see
   [Election Authority Key Ceremony] and [Ratification]).
2. **ACTIVE**: from the height $H_a$ at which the acknowledgement that
   completes ratification is recorded, until
   $\mathsf{vote}\_\mathsf{end}\_\mathsf{height}$. The **voting window**
   is the set of heights $h$ with $H_a < h \le \mathsf{vote}\_\mathsf{end}\_\mathsf{height}$;
   delegation and vote transactions are effective only when recorded
   within it.
3. **REVEALING**: the **reveal window** is the set of heights $h$ with
   $\mathsf{vote}\_\mathsf{end}\_\mathsf{height} < h \le \mathsf{reveal}\_\mathsf{end}\_\mathsf{height}$.
   The round's **final VCT root** is the root of its VCT after the last
   effective delegation or vote transaction recorded at or before
   $\mathsf{vote}\_\mathsf{end}\_\mathsf{height}$; it does not change
   thereafter. Share reveal transactions anchored to the final root
   (see [Vote Reveal Proof]) are effective only when recorded within
   the reveal window. Voters construct and submit their share reveal
   messages (see [Share Submission] and [Submission Timing]).
4. **TALLYING**: heights above
   $\mathsf{reveal}\_\mathsf{end}\_\mathsf{height}$. Partial decryption
   transactions are effective only when recorded in this state. The
   round has a tally once effective partial decryptions from at least
   $t$ trustees exist (see [Tally]); any party computes it at any later
   time, and the record never closes.
5. **CANCELLED**: a round for which an effective cancellation
   transaction exists (see [Cancellation]). No later transaction of the
   round is effective.
6. **FAILED**: a round that has not become ACTIVE by
   $\mathsf{vote}\_\mathsf{end}\_\mathsf{height}$, because its ceremony
   exhausted its attempts or some trustee never ratified it. No later
   transaction of the round is effective.

$\mathsf{reveal}\_\mathsf{end}\_\mathsf{height}$ MUST exceed
$\mathsf{vote}\_\mathsf{end}\_\mathsf{height}$ by at least $2\Delta$
(see [Deployment]), and $\mathsf{vote}\_\mathsf{end}\_\mathsf{height}$
MUST leave room for the ceremony (see [Poll Creation]). A deployment
MUST publish the reveal window length it uses, and SHOULD choose one
long enough that a voter who opens their wallet at ordinary intervals
does so at least once within it; see [Why a Separate Reveal Window].

Any number of voting rounds MAY exist in any state at once. All round
state — the VCT, the nullifier sets, the ceremony, the lifecycle and
the tally — is derived per round from the record, and nothing in this
specification requires the vote chain to know that any round exists.

### Proposals and Decisions

The voting process specified by this ZIP allows a poll runner to put
one or more questions — *proposals* — to eligible voters. For each
proposal, voters choose exactly one of a predefined set of labeled
*options*; the chosen option is the voter's *decision* for that
proposal.

#### Structure of a proposal

Each proposal has:

- A **title**, short and human-readable.
- An optional **description** providing additional context.
- Between 2 and $N_{\mathsf{opt}} = 8$ **options**, each carrying a
  human-readable label.
  Option labels MUST be non-empty ASCII strings.

Proposals in a voting round are assigned 1-indexed sequential
identifiers; options within a proposal are assigned 0-indexed
sequential indices. The encoding of a round's proposals, and the
$\mathsf{proposals}\_\mathsf{hash}$ that binds them into the round
identifier and the poll signature, are specified in
[Proposals Hash].

#### Decisions

A **decision** is a voter's chosen option for a specific proposal,
represented as the option's 0-indexed position within that proposal's
option list. A decision is never published; each share reveal carries
one ciphertext per option position, and the tally aggregates by
`(proposal_id, option position)`. See [Vote Reveal Proof] for the
construction and [Tally] for the aggregation.

#### Kinds of polls that can be expressed

A voting round can carry **1 to $\mathsf{MAX}\_\mathsf{PROPOSALS} = 50$
independent proposals**, and each proposal can offer **2 to 8 labeled
options**. This is sufficient for:

- Yes/no questions ("Approve proposal X?" with options Yes / No).
- Multiple-choice preference questions (for example, choosing among
  named candidates or funding tiers).
- Rating-style questions using a fixed option ladder.

The following ballot shapes are out of scope for this specification:

- Free-form write-in answers.
- Ranked-choice or weighted-ranking ballots.
- More than 50 proposals in a single voting round, or more than 8
  options in a single proposal.

The 50-proposal upper bound is imposed by the 51-bit vote authority
bitmask in the Vote Proof (1 bit is reserved as a sentinel; see
[Why Proposal Identifiers Start at 1]). Polls requiring more proposals
or richer ballot structures are split across multiple rounds or
expressed through an external layer.


### Snapshot Configuration

A voting round is anchored to a Zcash mainnet snapshot: a single
mainnet block, identified by both its height $H$ (the snapshot height)
and its hash $\mathsf{snapshot}\_\mathsf{blockhash}$, at which the
eligible Ironwood pool is captured. The poll runner chooses the block
subject to the following constraints:

- $H$ MUST be at or after the activation of the Ironwood pool.
- When the round is created (see [Poll Creation]), $H$ MUST be at
  least $\mathsf{min}\_\mathsf{confirmations}$ blocks below the tip of
  the Zcash mainnet best chain, where
  $\mathsf{min}\_\mathsf{confirmations}$ is a deployment parameter (see
  [Deployment]).
- $\mathsf{snapshot}\_\mathsf{blockhash}$ MUST be the hash of the block
  at height $H$ on the Zcash mainnet best chain.

The confirmation depth keeps the snapshot out of the part of the chain
that is realistically subject to reorganisation. The block hash binds
the round to a block rather than to a height, so that a reorganisation
affecting the snapshot is detected rather than silently changing the
eligible note set. If the block at height $H$ on the best chain ceases
to have hash $\mathsf{snapshot}\_\mathsf{blockhash}$ after the round is
created, the round is no longer well formed under
[Reading the Snapshot Roots]: the snapshot it is anchored to no longer
exists, and the eligibility of every vote cast in it is undefined. No
transaction is needed to abandon it; a trustee MUST NOT ratify or
decrypt such a round, a wallet MUST NOT take part in it, and a verifier
treats it as having no result. The poll runner creates a new round.

Choosing the snapshot is the start of round setup, not a single
automatic action. The poll runner is responsible for the following
coordinated activities:

1. **Read the snapshot roots.** The Ironwood pool note commitment
   tree root ($\mathsf{nc}\_\mathsf{root}$) and the nullifier non-membership
   tree root ($\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$) are
   the two roots $\mathsf{rt^{cm}}$ and $\mathsf{rt^{excl}}$ of the pool
   snapshot at height $H$, as defined in the "Pool Snapshot" section of
   `draft-valargroup-orchard-balance-proof` [^balance-proof], on
   the chain whose block at height $H$ has hash
   $\mathsf{snapshot}\_\mathsf{blockhash}$. No party has discretion over
   their values. The poll runner reads them from a Zcash consensus node
   as specified in [Reading the Snapshot Roots].

2. **Ensure the nullifier service has the snapshot's PIR
   database.** Where the deployment offers a nullifier service, the
   poll runner coordinates with each nullifier service operator (see
   the "Nullifier Service Operator" section of
   `draft-valargroup-shielded-voting-setup` [^voting-setup]) so that
   their ingest and export pipelines (see the "Nullifier Service"
   section of that document) have run to the chosen height before the
   round opens, so wallets can retrieve exclusion proofs against that
   snapshot by private information retrieval
   (`draft-valargroup-nullifier-pir` [^nullifier-pir]).

3. **Carry the values in the round creation transaction.**
   $\mathsf{nc}\_\mathsf{root}$ and
   $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ are fields of
   the round creation transaction (see [Poll Creation]), so that every
   verifier and wallet client uses the same values as ZKP public
   inputs.

### Reading the Snapshot Roots

Both snapshot roots are properties of Zcash mainnet state at
$(H, \mathsf{snapshot}\_\mathsf{blockhash})$. A party obtains them by
reading them from a Zcash consensus node that has validated the chain
to at least that height. No other party's construction of either tree
is a source for the round's roots.

1. Confirm that the block at height $H$ on the Zcash consensus node's
   best chain has hash $\mathsf{snapshot}\_\mathsf{blockhash}$. If it
   does not, the round is not well formed.
2. Obtain $\mathsf{nc}\_\mathsf{root}$: the Ironwood pool note
   commitment tree root as of the end of block $H$. This is Zcash
   consensus data; a Zcash consensus node computes it while validating
   the chain and exposes it as the pool's anchor at that height.
3. Obtain $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$: the root
   of the nullifier non-membership tree over every Ironwood pool
   nullifier revealed at or before $H$, constructed as specified in
   `draft-valargroup-orchard-balance-proof` [^balance-proof].
   Zcash blocks do not commit to this tree; a Zcash consensus node
   maintains it as an index over the nullifier set it already tracks,
   and serves its root.
4. Compare both values with the round's $\mathsf{nc}\_\mathsf{root}$
   and $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$.

A round is **well formed** if all four steps succeed.

Any Zcash consensus node that follows the Zcash protocol and serves
both roots is acceptable; parties need not agree on an implementation.
The construction in `draft-valargroup-orchard-balance-proof`
[^balance-proof] is the sole authority on the value of
$\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$, and a Zcash consensus
node that serves it MUST implement that construction.

**Who performs it.** The poll runner MUST perform it before creating a
round. Each trustee MUST perform it, against a Zcash consensus node
under its own control, before ratifying a round (see [Ratification]).
Each wallet MUST perform it, against the Zcash consensus node it
trusts for Zcash state, before taking part in a round, as specified in
`draft-valargroup-shielded-voting-wallet-api` [^wallet-api]. Agreement
with another party's copy of the round does not satisfy this for any
of them: comparing two copies of the same values establishes only that
two parties received the same input, not that the input is correct.

### Poll Creation

Any account MAY create a voting round by recording a round creation
transaction (see [Round Creation Transaction]). The transaction carries:

- $\mathsf{creator}\_\mathsf{pk}$, the Ed25519 public key under which
  the transaction is signed; its holder is the round's **creator**.
- `snapshot_height` and `snapshot_blockhash` (see
  [Snapshot Configuration]).
- `nc_root` and `nullifier_imt_root`, the snapshot roots (see
  [Reading the Snapshot Roots]).
- `vote_end_height` and `reveal_end_height`, vote chain heights (see
  [Round Lifecycle]).
- `stage_window` and `max_attempts`, the ceremony's timing parameters
  (see [Election Authority Key Ceremony]).
- `proposals`, the round's proposals (see [Proposals and Decisions]).
- `trustees`, the round's trustees in order: for each, its trustee
  account key and its trustee ceremony key.
- `title` and `description`, human-readable.

Every party derives the remaining facts about the round from the
record: $\mathsf{proposals}\_\mathsf{hash}$ and
$\mathsf{trustees}\_\mathsf{hash}$ from the transaction's fields (see
[Proposals Hash] and [Trustees Hash]),
$\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ by
[Voting Round Identifier], the creation height $H_c$ from where the
transaction is recorded, the election authority public key from the
ceremony, and the state from [Round Lifecycle].

A round creation transaction is effective if and only if:

1. it is well formed per [Transaction Formats], and its signature
   verifies under $\mathsf{creator}\_\mathsf{pk}$;
2. `proposals` satisfies [Proposals and Decisions];
3. `trustees` names at least two trustees, with distinct trustee
   account keys;
4. `reveal_end_height` $\ge$ `vote_end_height` $+ 2\Delta$;
5. `vote_end_height` $\ge H_c + 5 \cdot$ `max_attempts` $\cdot$
   `stage_window`, so that every ceremony attempt can complete before
   voting must end, and `stage_window` $\ge 1$ and `max_attempts`
   $\ge 1$;
6. no earlier effective round creation transaction has the same
   $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$.

Nothing in the record verifies the snapshot roots; see
[Reading the Snapshot Roots] for who does. Creating a round requires
no privileged role: any well-formed round is recorded, and a round
that no trustee ratifies never opens. The creator has one power over
the round beyond having named its parameters: it MAY cancel the round
while it is pending (see [Cancellation]).

The round enters PENDING at $H_c$, and the key ceremony begins in the
following block (see [Election Authority Key Ceremony]). Once the
ceremony completes and every trustee has ratified the round (see
[Ratification]), the round is ACTIVE and the voting window is open.
The round enters REVEALING after `vote_end_height` and TALLYING after
`reveal_end_height` (see [Round Lifecycle]). Clients construct their
share submission schedule within the reveal window, as specified in
[Submission Timing]. There is no last-moment buffer: a client near the
deadline compresses its schedule rather than concentrating its weight.

### Poll Signature

The poll runner publishes a round to wallets as a vote configuration,
as specified in `draft-valargroup-shielded-voting-wallet-api`
[^wallet-api], and signs it. A wallet accepts a configuration only if
it carries a valid signature from a poll runner key the wallet
recognises.

The bytes covered by the signature are the concatenation, in this
order, of:

| Component | Width |
|---|---|
| The ASCII string `ZcashVotingPollSignature:v4` | 27 bytes |
| `vote_round_id` | 32 bytes |
| `snapshot_height`, big-endian unsigned | 8 bytes |
| `snapshot_blockhash` | 32 bytes |
| `nc_root` | 32 bytes |
| `nullifier_imt_root` | 32 bytes |
| `proposals_hash` | 32 bytes |
| `vote_end_height`, big-endian unsigned | 8 bytes |
| `reveal_end_height`, big-endian unsigned | 8 bytes |
| `trustees_hash` | 32 bytes |

where `proposals_hash` and `trustees_hash` are as specified in
[Proposals Hash] and [Trustees Hash]. All components are fixed width,
so the encoding is unambiguous without length prefixes. The domain
separator distinguishes these bytes from any other signature the same
key may produce, and its version is that of the vote configuration
format that carries the signature.

The signature establishes that the round — its snapshot, its proposals,
its deadlines and its trustees — is the one the poll runner is running.
It does not establish that the snapshot roots are correct or that the
election authority key is genuine, and a wallet MUST NOT treat it as
doing so. A wallet verifies those two things itself:

- **Snapshot roots.** The wallet MUST perform
  [Reading the Snapshot Roots] against the Zcash consensus node it
  trusts, and MUST NOT take part in a round whose roots differ from
  what that node serves. The same node is the wallet's oracle for every
  other fact about Zcash state, so this adds no trust the wallet did not
  already extend.
- **Election authority key.** $\mathsf{ea}\_\mathsf{pk}$ exists only
  once the ceremony completes and is not covered by the signature. The
  wallet derives it from the record, or reads it from a vote chain
  node, and MUST verify that every trustee listed in the configuration
  has an effective acknowledgement transaction committing to that
  $\mathsf{ea}\_\mathsf{pk}$ and this
  $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ (see
  [Election Authority Key Ceremony]), signed by that trustee's account
  key. A key acknowledged by every trustee is the key those trustees
  hold shares of; a key that is not is one the wallet MUST NOT encrypt
  to.

Because the signature does not cover $\mathsf{ea}\_\mathsf{pk}$, the
poll runner can publish and sign a configuration as soon as the round
is created, and wallets can authenticate it before the round opens.

### Election Authority Key Ceremony

Each voting round uses a fresh election authority keypair
$(\mathsf{ea}\_\mathsf{sk}, \mathsf{ea}\_\mathsf{pk})$. The ceremony
that produces it is a distributed key generation (DKG) among the
round's trustees, run over the vote chain, which serves as the
ceremony's authenticated, ordered broadcast channel: each trustee
records its contribution in a transaction signed by its trustee account
key, and every party reads the same record. The construction is
Pedersen's DKG [^pedersen-dkg] with Feldman verifiable secret sharing
[^feldman] and a proof of knowledge of each contribution, in the form
analysed in [^gjkr] and adopted by FROST [^frost].

No party holds $\mathsf{ea}\_\mathsf{sk}$ at any point. Each trustee
contributes a random polynomial; the key is the sum of the constant
terms, and each trustee's share is the sum of every trustee's
evaluation at its index. Scoping the key to one voting round bounds
the damage from a share compromise to that round, and means a trustee
that leaves the set cannot decrypt later rounds. The cryptographic
constructions used below are specified in
[El Gamal Encryption on Pallas], [ECIES on Pallas] and
[Chaum-Pedersen DLEQ Proofs]; the transactions are specified in
[Transaction Formats].

**Participants.** The round's trustees are the parties named in its
round creation transaction (see [Poll Creation]), each by its trustee
account key, under which it signs its ceremony, acknowledgement,
cancellation and partial decryption transactions, and its trustee
ceremony key, a Pallas public key under which the shares dealt to it
are encrypted. Let $n$ be their number and index them $1, \ldots, n$
in the order the round creation transaction lists them; a trustee's
index is its evaluation point. The round's decryption threshold is
$t = \lceil n/2 \rceil + 1$, with a minimum of 2. How trustees are
selected and how the poll runner obtains their keys is specified in
`draft-valargroup-shielded-voting-setup` [^voting-setup]. A trustee
MUST NOT be a validator of the vote chain, for any round on it.

**Timing.** The ceremony runs on a fixed grid of vote chain heights
set by the round creation transaction. An attempt consists of five
stages, each occupying a window of `stage_window` blocks; the round
allows up to `max_attempts` attempts. Stage $j \in \{1 \ldots 5\}$ of
attempt $k \in \{1 \ldots$ `max_attempts`$\}$ is the set of heights
$h$ with

$$H_c + \bigl((k-1) \cdot 5 + (j-1)\bigr) \cdot \texttt{stage\_window} < h \le H_c + \bigl((k-1) \cdot 5 + j\bigr) \cdot \texttt{stage\_window}$$

Every ceremony transaction names its attempt, and is effective only
if recorded within the window of its stage in that attempt, only if
every earlier stage of that attempt completed, and only if no earlier
attempt completed and no cancellation has been recorded. A stage
**completes** when every trustee's required transaction for it is
effective; a stage in whose window some trustee's required
transaction is not recorded has **failed**, and with it the attempt.
The next attempt, if any remain, begins at its own grid position with
fresh randomness and the same trustees; the record shows which
trustee did not publish.

**Stage 1: commitment.** Each trustee $i$:

1. Samples $t$ coefficients $a_{i,0}, \ldots, a_{i,t-1}$ uniformly at
   random from the Pallas scalar field, defining
   $f_i(x) = \sum_{k=0}^{t-1} a_{i,k}\, x^k$.
2. Computes the Feldman commitments $A_{i,k} = [a_{i,k}]\, G$ for
   $k = 0, \ldots, t-1$.
3. Computes a Schnorr proof of knowledge of $a_{i,0}$: samples $k_i$
   at random, sets $R_i = [k_i]\, G$,
   $c_i = \mathsf{BLAKE2b\text{-}256}(\texttt{"svote-dkg-pok-v1"} \mathbin\| \mathsf{voting}\_\mathsf{round}\_\mathsf{id} \mathbin\| \mathsf{attempt} \mathbin\| i \mathbin\| \mathsf{repr}(A_{i,0}) \mathbin\| \mathsf{repr}(R_i))$
   reduced to a Pallas scalar, where $\mathsf{attempt}$ and $i$ are
   single bytes, and $\mu_i = k_i + c_i \cdot a_{i,0}$.
4. Records a ceremony commitment transaction (see
   [Ceremony Commitment Transaction]) carrying $A_{i,0}, \ldots,
   A_{i,t-1}$ and $(R_i, \mu_i)$.

A ceremony commitment transaction is effective only if its proof of
knowledge verifies, that is, if
$[\mu_i]\, G = R_i + [c_i]\, A_{i,0}$, and only if it is the first
from that trustee in that attempt. The proof of knowledge prevents
a trustee from choosing its commitment as a function of others' and so
biasing or cancelling the key; see [Why Distributed Key Generation].

**Stage 2: dealing.** Each trustee $i$ computes $f_i(j)$ for every
other trustee $j$, encrypts each to trustee $j$'s ceremony key using
ECIES with a fresh ephemeral scalar per recipient, and records the
encrypted shares in a ceremony dealing transaction (see
[Ceremony Dealing Transaction]). A trustee MUST then erase
$a_{i,1}, \ldots, a_{i,t-1}$ and every $f_i(j)$ for $j \neq i$,
retaining only $f_i(i)$.

**Stage 3: complaints.** Each trustee $j$ decrypts each share $f_i(j)$
addressed to it and checks it against trustee $i$'s commitments:

$$[f_i(j)]\, G = \sum_{k=0}^{t-1} [j^k]\, A_{i,k}$$

If the check fails for some $i$, or the share cannot be decrypted,
trustee $j$ records a complaint transaction against $i$ (see
[Complaint Transaction]). No transaction is required of a trustee that
has no complaint, and Stage 3 always completes.

**Stage 4: responses.** A trustee $i$ against which an effective
complaint from $j$ exists MUST record a complaint response transaction
(see [Complaint Response Transaction]) publishing $f_i(j)$ in the
clear. A response is effective only if the published value satisfies
the equation above against $A_{i,\cdot}$. Stage 4 completes when every
effective complaint has an effective response; if any does not, the
attempt has failed. The complaining trustee uses the published value
as its share from $i$.

**Key derivation.** On the completion of Stage 4, every party computes
from the record:

$$\mathsf{ea}\_\mathsf{pk} = \sum_{i=1}^{n} A_{i,0}$$

$$\mathsf{pk}_j = \sum_{i=1}^{n} \sum_{k=0}^{t-1} [j^k]\, A_{i,k} \quad \text{for each } j \in \{1 \ldots n\}$$

Each trustee $j$ computes its key share

$$\mathsf{sk}_j = \sum_{i=1}^{n} f_i(j)$$

and MUST verify that $[\mathsf{sk}_j]\, G = \mathsf{pk}_j$ before
acknowledging. The shares $\mathsf{sk}_j$ are Shamir shares [^shamir]
of $\mathsf{ea}\_\mathsf{sk} = \sum_i a_{i,0}$ on the degree-$(t-1)$
polynomial $\sum_i f_i$, so [Tally] applies to them unchanged. Every
trustee share key $\mathsf{pk}_j$ is computed from recorded values and
can be recomputed by any party.

**Stage 5: acknowledgement.** Each trustee $j$ that has verified its
share, and has verified the round's snapshot roots as [Ratification]
requires, records an acknowledgement transaction (see
[Acknowledgement Transaction]) carrying $\mathsf{ea}\_\mathsf{pk}$ and
signed by its trustee account key. An acknowledgement is effective
only if its $\mathsf{ea}\_\mathsf{pk}$ equals the value derived above.
A trustee MUST NOT acknowledge a share that fails the share key check.
Committing to $\mathsf{ea}\_\mathsf{pk}$ keeps an acknowledgement from
carrying over to a rekeyed attempt; the transaction's round identifier
keeps it from carrying over to another round. This acknowledgement is
the trustee's ratification of the round; see [Ratification]. Wallets
verify these acknowledgements to authenticate
$\mathsf{ea}\_\mathsf{pk}$; see [Poll Signature].

**Completion.** The ceremony completes, and the round becomes ACTIVE,
at the height at which the last trustee's acknowledgement in a
completed attempt is recorded. Ceremony transactions of any later
attempt are not effective.

**Failure.** An attempt whose Stage 1, 2, 4 or 5 fails is followed by
the next attempt on the grid. When `max_attempts` attempts have failed
the round is FAILED (see [Round Lifecycle]), and the poll runner MAY
create a new round with an amended trustee set. A pending round MAY
also be cancelled at any time (see [Cancellation]), which is the
prompt path when a trustee is known not to be taking part.

**Trustee set changes.** A round's trustee set is fixed at creation. A
trustee cannot be added to a round in progress, and a trustee that
leaves retains its share and cannot be compelled to delete it;
per-round keys bound what that share is worth, since it opens nothing
in any other round.

### Ratification

The poll signature ([Poll Signature]) establishes that a round is the
one its poll runner is running. It does not establish that the round
will be tallied. Those are different parties: the poll runner
configures a round, and the trustees decrypt its result. A round can be
correctly configured, voted in, and never opened.

A trustee ratifies a round with the acknowledgement it records at the
end of the key ceremony, as specified in
[Election Authority Key Ceremony]. Recording it is the trustee's
statement that it has read the round's snapshot roots from a Zcash
consensus node under its own control and found them correct (see
[Reading the Snapshot Roots]), that it holds a verified share for the
round, and that it will take part in the round's tally. A trustee MUST
NOT ratify a round whose snapshot roots it has not verified.

A round is not ACTIVE until every trustee named in it has ratified it.
Ratification is unanimous rather than a threshold: a round ratified by
only $t$ of its $n$ trustees would be lost if any one of them became
unavailable during the round, whereas a round ratified by all $n$ can
lose up to $n - t$ of them and still be tallied. Trustees that cannot
agree unanimously to serve a round should not be that round's
trustees. Because ratifications are vote chain transactions, they are
recorded with the round.

A ratification is a statement of intent, not an enforceable commitment.
A trustee can ratify and then decline to take part, and nothing in this
document prevents that. What ratification provides is that the decision
is made and recorded before voters commit to the round, rather than
discovered afterwards, and that a trustee declining to tally a round it
ratified is visibly departing from a recorded statement. See
[Why Ratification Precedes Voting].

### Cancellation

A pending round can be withdrawn before it opens. Any trustee of the
round, or its creator, MAY record a cancellation transaction (see
[Cancellation Transaction]), signed by its trustee account key or by
$\mathsf{creator}\_\mathsf{pk}$ respectively. A cancellation is
effective if and only if the round is PENDING at the height at which
it is recorded. From that height the round is CANCELLED: no ceremony,
acknowledgement, delegation, vote, share reveal or partial decryption
transaction of the round is effective, whatever attempt or stage it
belongs to.

Cancellation exists so that a round stalled by a trustee that will not
take part can be replaced at once, rather than after every attempt on
the ceremony grid has run out, and so that a late acknowledgement
cannot open a round its poll runner has already replaced. It confers
no power a trustee does not already have: any trustee can keep a round
from opening by not ratifying it, and cancellation makes that outcome
immediate and recorded. It is not available once a round is ACTIVE,
where it would be a new power; a single trustee cannot block a tally,
since ratification is unanimous and decryption needs only $t$
trustees.

### Election Authority Key Custody

**Share generation.** The ceremony in [Election Authority Key Ceremony]
generates the key in distributed form: no party ever holds
$\mathsf{ea}\_\mathsf{sk}$, and every trustee can verify its own share
against recorded commitments without trusting any other participant.
The claim that no single party holds the key therefore rests on the
ceremony's construction rather than on any party's promise to erase
anything. What each trustee MUST erase is its own polynomial's
non-constant coefficients and the shares it dealt to others, as
specified in Stage 2; retaining them does not expose the key, but does
expose other trustees' shares from that trustee.

**Retention.** Each trustee MUST destroy its share once the round has
a tally and the trustee has published its partial decryption, and a
deployment MUST publish the retention period it applies. The encrypted
shares of every individual vote remain on the vote chain permanently,
and their encryption is not post-quantum, so retained shares are a live
capability against a permanent record of individual voters' share
values, not a dormant convenience. Retention is not needed for audit:
the partial decryptions and their DLEQ proofs are recorded and can be
re-verified at any time without the key. A deployment that retains
shares nonetheless MUST state for how long, and MUST treat that period
as the period over which its amount-privacy claims hold.


## Vote Chain Record

The vote chain is an ordered, append-only record of transactions, each
recorded at a block height. Its validators determine which
transactions are recorded and in what order, and nothing else: the
vote chain applies none of the rules in this ZIP, maintains no state
this ZIP defines, and rejects no transaction on this ZIP's account
(see [Why the Vote Chain Validates Nothing]). Every rule in this ZIP
is a rule for interpreting the record, and every party that reads the
record — a wallet, a trustee, a tallier, an auditor, or a vote chain
node offering a query interface — applies the same rules to the same
record and derives the same result.

What a vote chain transaction's envelope contains — the fee, the
account that pays it, the chain identifier — is the chain's own
matter and is out of scope (see [Non-requirements]). This ZIP
specifies the payload each transaction carries, in
[Transaction Formats].

### Effective Transactions

A recorded transaction is **effective** if it is well formed per
[Transaction Formats] and satisfies every rule this ZIP states for its
type, evaluated against the derived state of its round as of the
effective transactions recorded before it. Transactions are evaluated
in record order: by height, and within a block in the order the block
lists them. A transaction that is not effective is recorded like any
other, has no effect on any derived state, and is ignored by every
rule; it is not an error, and nothing distinguishes a party that
recorded one from a party that recorded nothing.

The rules by type are stated where the type is specified:

- round creation: [Poll Creation];
- cancellation: [Cancellation];
- ceremony commitment, dealing, complaint, complaint response and
  acknowledgement: [Election Authority Key Ceremony];
- delegation, vote and share reveal: the "Out-of-Circuit Verification"
  subsections of [Delegation Proof], [Vote Proof] and
  [Vote Reveal Proof];
- partial decryption: [Partial Decryption].

Where two recorded transactions conflict — two round creations with
one identifier, two delegations publishing one governance nullifier,
two acknowledgements from one trustee in one attempt — the earlier in
record order is the one that can be effective, and the later is not.

From the effective transactions, every party derives, per round:

- the round's parameters and identifier, from its round creation
  transaction;
- its lifecycle state at every height (see [Round Lifecycle]);
- the ceremony's progress, $\mathsf{ea}\_\mathsf{pk}$ and every trustee
  share key $\mathsf{pk}_i$ (see [Election Authority Key Ceremony]);
- the **Vote Commitment Tree** as defined in [Vote Commitment Tree],
  its root after every height, and its final root;
- the three **nullifier sets** as defined in [Nullifier Sets];
- the aggregate ciphertexts and the tally (see [Tally]).

A vote chain node MAY maintain any of this as an index and serve it
over a query interface (see
`draft-valargroup-shielded-voting-wallet-api` [^wallet-api]). A party
that takes such an answer from a node it does not control has not
verified it; what it can lose by doing so is bounded by the rules
above, since a proof anchored to a wrong root is simply not effective,
and a party that wants the assurance derives the state itself.

### Transaction Inclusion

The vote chain is a CometBFT chain, and its validators determine which
transactions are recorded. This section states the consequences for a
voting round.

**What validators do.** Validators record transactions. They apply
none of this ZIP's rules and need not implement any of them; a
transaction this ZIP treats as ineffective is recorded on the same
terms as any other, and the chain's own admission rules — a parseable
envelope, a fee — are the only ones a submitter faces. A validator
cannot reject a transaction as invalid under this ZIP, and so cannot
present exclusion as validation.

**What validators can do.** Validators controlling enough stake to
control block production can decline to record transactions. A share
reveal exposes its proposal identifier but not its decision (see
[Vote Reveal Proof]), and per-option totals are not public while a
round is open, so validators acting alone cannot select reveals to
exclude by the option they support. They can exclude reveals by
proposal, by time of arrival, by network origin, or wholesale. They
can likewise withhold a trustee's ceremony or partial decryption
transactions, delaying or preventing a round from opening or being
tallied. A coalition of validators and $t$ trustees could decrypt
reveals as they arrive and exclude by option; the requirement that the
two sets be disjoint (see [Requirements] and
[Election Authority Key Ceremony]) exists to keep that coalition from
being a single organisation.

**What validators cannot do.** They cannot create votes, alter the
weight of a recorded vote, or misreport the tally: each is prevented
by a proof or a derivation that any party can check from the record.
Exclusion only removes support; it cannot manufacture it.

**What this means for a result.** A published result is a lower bound
on the support each option received, not a measurement of it. Results
SHOULD be described in those terms.

Separately from what validators can do, whether a result may be
described as representative of coinholder sentiment also depends on
conditions on the round itself, such as the diversity of wallet
implementations available to voters. Those conditions are not yet
specified; see [Open issues].

**Detection.** Exclusion is detectable but not provable from the record
alone. An excluded transaction leaves no trace in the record that
excluded it. Available signals are: a count of effective votes against
the count of effective share reveals; the contents of honest vote chain
nodes' mempools, compared with what was subsequently recorded; and
voters observing that their own shares never appeared. The last is
currently unavailable in practice, because a voter querying a node for
their own share nullifiers reveals which nullifiers are theirs; see
[Open issues].

A deployment SHOULD publish, for each round, the count of share reveal
transactions accepted into the mempool alongside the count recorded in
blocks, from more than one operator, so that a discrepancy is visible
without requiring any party to be trusted.

**Threshold.** A deployment MUST publish the stake distribution across
validators for a round, and the proportion of stake required to control
block production, so that the size of the coalition required to exclude
transactions is a published figure rather than an inferred one.


## Transaction Formats

This section specifies the payload of every transaction this ZIP
defines. A payload is the byte string a vote chain transaction carries
for this protocol; how the chain wraps it is out of scope. Every party
that interprets the record parses payloads by these rules, and a
payload that does not parse exactly — with no bytes left over and
every field within its stated domain — is not effective.

### Encoding

Payloads are built from the following primitive encodings.

| Type | Encoding |
|---|---|
| `u8`, `u16`, `u32`, `u64` | Unsigned integer, little-endian, of the stated width. |
| `compactSize` | A variable-length unsigned integer as specified for Zcash transactions in ZIP 225 [^zip-0225]. |
| `bytes[N]` | Exactly $N$ bytes. |
| `bytes` | A `compactSize` length $\ell$ followed by $\ell$ bytes. |
| `string` | A `bytes` whose content is UTF-8 with no NUL byte. |
| `fe` | A Pallas base field element: 32 bytes, little-endian, canonical [^protocol-pallasandvesta]. A value not less than the modulus does not parse. |
| `scalar` | A Pallas scalar field element: 32 bytes, little-endian, canonical. A value not less than the modulus does not parse. |
| `point` | A Pallas point in the 32-byte compressed encoding $\mathsf{repr}$ [^protocol-pallasandvesta]. A value that does not decode to a point does not parse. |
| `ciphertext` | An El Gamal ciphertext: `point` $C_1$ followed by `point` $C_2$, 64 bytes. |
| `proof` | A `bytes` holding a Halo 2 proof [^halo2] for the circuit the transaction type names. |
| `spendauthsig` | A $\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$ signature, 64 bytes [^protocol-concretespendauthsig]. |
| `ed25519pk` | An Ed25519 public key, 32 bytes [^rfc8032]. |
| `ed25519sig` | An Ed25519 signature, 64 bytes [^rfc8032]. |
| `vec<T>` | A `compactSize` count $m$ followed by $m$ encodings of `T`. |

Fields are concatenated in the order listed for each type, with no
padding. Every payload begins with a `tx_type` (`u8`) that names its
type, from the table below, and every payload but a round creation
continues with `voting_round_id` (`bytes[32]`), the identifier of the
round it belongs to, as a little-endian field element.

| `tx_type` | Transaction |
|---|---|
| 1 | [Round Creation Transaction] |
| 2 | [Cancellation Transaction] |
| 3 | [Ceremony Commitment Transaction] |
| 4 | [Ceremony Dealing Transaction] |
| 5 | [Complaint Transaction] |
| 6 | [Complaint Response Transaction] |
| 7 | [Acknowledgement Transaction] |
| 8 | [Delegation Transaction] |
| 9 | [Vote Transaction] |
| 10 | [Share Reveal Transaction] |
| 11 | [Partial Decryption Transaction] |

### Signed Payloads

Transactions recorded by the creator or by a trustee end with a
`signature` (`ed25519sig`) by the signer's Ed25519 key — the trustee
account key, or $\mathsf{creator}\_\mathsf{pk}$ — over the byte
string

$$\texttt{"ZcashVoteTx:v1"} \mathbin\| \mathsf{prefix}$$

where $\mathsf{prefix}$ is every byte of the payload preceding the
`signature` field, including `tx_type`. Because the prefix carries the
type, the round identifier and, for ceremony transactions, the attempt
and trustee index, a signature cannot be replayed as a different
transaction. A payload whose signature does not verify under the key
the rules name for it is not effective. Delegation, vote and share
reveal transactions carry no such signature: each is authorised by the
proof and, where stated, the spend authorization signature it carries,
and is valid regardless of who records it.

### Proposals Hash

$\mathsf{proposals}\_\mathsf{hash}$ is the BLAKE2b-256 [^blake2] hash,
with the 16-byte personalization `ZcashVoteProposl`, of the encoding
of the `proposals` field of the round creation transaction, exactly as
that field appears in the payload (a `vec<Proposal>`; see
[Round Creation Transaction]). A wallet that receives proposals in
another form, such as the vote configuration's JSON, MUST re-encode
them by these rules to compute the hash.

### Trustees Hash

$\mathsf{trustees}\_\mathsf{hash}$ is the BLAKE2b-256 [^blake2] hash,
with the 16-byte personalization `ZcashVoteTrustee`, of the
concatenation of the trustees' account keys, 32 bytes each, in the
order the round creation transaction lists them, without length
prefixes. For $n$ trustees the input is exactly $32n$ bytes. Ceremony
keys do not enter the hash: the round identifier binds the trustee set
by the keys under which trustees ratify, and a wallet identifies a
trustee's acknowledgement by its account key.

### Round Creation Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 1 |
| `creator_pk` | `ed25519pk` | The creator's key, $\mathsf{creator}\_\mathsf{pk}$ |
| `snapshot_height` | `u64` | Zcash mainnet snapshot height |
| `snapshot_blockhash` | `bytes[32]` | Hash of the snapshot block, in the byte order of the Zcash block header |
| `nc_root` | `fe` | Ironwood pool note commitment tree root at the snapshot |
| `nullifier_imt_root` | `fe` | Nullifier non-membership tree root at the snapshot |
| `vote_end_height` | `u64` | Last height of the voting window |
| `reveal_end_height` | `u64` | Last height of the reveal window |
| `stage_window` | `u32` | Blocks per ceremony stage |
| `max_attempts` | `u8` | Ceremony attempts allowed |
| `proposals` | `vec<Proposal>` | The round's proposals, 1 to 50 entries |
| `trustees` | `vec<Trustee>` | The round's trustees, at least 2 entries |
| `title` | `string` | At most 256 bytes |
| `description` | `string` | At most 4096 bytes |
| `signature` | `ed25519sig` | By `creator_pk`; see [Signed Payloads] |

A `Proposal` is:

| Field | Type | Description |
|---|---|---|
| `id` | `u8` | 1-indexed; the $k$-th proposal listed MUST have `id` $= k$ |
| `title` | `string` | At most 256 bytes |
| `description` | `string` | At most 4096 bytes, MAY be empty |
| `options` | `vec<Option>` | 2 to $N_{\mathsf{opt}}$ entries |

An `Option` is:

| Field | Type | Description |
|---|---|---|
| `index` | `u8` | 0-indexed; the $k$-th option listed MUST have `index` $= k - 1$ |
| `label` | `string` | Non-empty ASCII, at most 64 bytes |

A `Trustee` is:

| Field | Type | Description |
|---|---|---|
| `account_pk` | `ed25519pk` | The trustee account key |
| `ceremony_pk` | `point` | The trustee ceremony key |

The rules for effectiveness are in [Poll Creation].

### Cancellation Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 2 |
| `voting_round_id` | `bytes[32]` | |
| `signer` | `u8` | 0 for the creator; otherwise the trustee's index, 1 to $n$ |
| `signature` | `ed25519sig` | By `creator_pk` if `signer` is 0, else by trustee `signer`'s account key |

The rules for effectiveness are in [Cancellation].

### Ceremony Commitment Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 3 |
| `voting_round_id` | `bytes[32]` | |
| `attempt` | `u8` | 1 to `max_attempts` |
| `trustee` | `u8` | The recording trustee's index $i$, 1 to $n$ |
| `commitments` | `vec<point>` | $A_{i,0}, \ldots, A_{i,t-1}$; exactly $t$ entries |
| `pok_R` | `point` | $R_i$ |
| `pok_mu` | `scalar` | $\mu_i$ |
| `signature` | `ed25519sig` | By trustee $i$'s account key |

### Ceremony Dealing Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 4 |
| `voting_round_id` | `bytes[32]` | |
| `attempt` | `u8` | |
| `trustee` | `u8` | The dealing trustee's index $i$ |
| `shares` | `vec<EncryptedShare>` | Exactly $n - 1$ entries, for recipients $j \ne i$ in ascending order of $j$ |
| `signature` | `ed25519sig` | By trustee $i$'s account key |

An `EncryptedShare` is the ECIES encryption (see [ECIES on Pallas]) of
the 32-byte little-endian encoding of $f_i(j)$ to trustee $j$'s
ceremony key:

| Field | Type | Description |
|---|---|---|
| `ephemeral_pk` | `point` | The ephemeral public key $E$ |
| `ciphertext` | `bytes[48]` | ChaCha20-Poly1305 ciphertext (32 bytes) and tag (16 bytes) |

### Complaint Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 5 |
| `voting_round_id` | `bytes[32]` | |
| `attempt` | `u8` | |
| `trustee` | `u8` | The complaining trustee's index $j$ |
| `against` | `u8` | The index $i \ne j$ of the trustee whose share failed |
| `signature` | `ed25519sig` | By trustee $j$'s account key |

### Complaint Response Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 6 |
| `voting_round_id` | `bytes[32]` | |
| `attempt` | `u8` | |
| `trustee` | `u8` | The responding trustee's index $i$ |
| `to` | `u8` | The complaining trustee's index $j$ |
| `share` | `scalar` | $f_i(j)$ in the clear |
| `signature` | `ed25519sig` | By trustee $i$'s account key |

### Acknowledgement Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 7 |
| `voting_round_id` | `bytes[32]` | |
| `attempt` | `u8` | |
| `trustee` | `u8` | The acknowledging trustee's index $j$ |
| `ea_pk` | `point` | The election authority public key the trustee holds a share of |
| `signature` | `ed25519sig` | By trustee $j$'s account key |

A wallet verifies an acknowledgement from the payload alone: it checks
that `ea_pk` equals the round's key, that `trustee` names the trustee
whose account key it is checking, and that `signature` verifies under
that key over the bytes specified in [Signed Payloads]. The rules for
effectiveness are in [Election Authority Key Ceremony].

### Delegation Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 8 |
| `voting_round_id` | `bytes[32]` | |
| `proof` | `proof` | The Delegation Proof $\pi\_\mathsf{del}$ |
| `sighash_del` | `bytes[32]` | Client-computed sighash (see [Delegation Sighash]) |
| `sigma_del` | `spendauthsig` | $\sigma\_\mathsf{del}$, under $\mathsf{rk}$ |
| `signed_note_nullifier` | `fe` | Dummy note nullifier |
| `rk` | `point` | Randomized verification key |
| `rt_cm` | `fe` | Note commitment tree root |
| `rt_excl` | `fe` | Non-membership tree root |
| `van` | `fe` | VAN commitment |
| `gov_null` | `fe` $\times 5$ | Governance nullifiers $\mathsf{gov}\_{\mathsf{null}\_1} \ldots \mathsf{gov}\_{\mathsf{null}\_5}$, in slot order |
| `cmx_new` | `fe` | Output note commitment |

The public inputs of the proof are the fields above together with
`voting_round_id`. The rules for effectiveness are in
[Delegation Proof].

### Vote Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 9 |
| `voting_round_id` | `bytes[32]` | |
| `proof` | `proof` | The Vote Proof $\pi_{\text{vote}}$ |
| `sigma_vote` | `spendauthsig` | $\sigma_{\text{vote}}$, under $\mathsf{r}_{\mathsf{vpk}}$ |
| `van_nullifier` | `fe` | Old VAN nullifier |
| `r_vpk` | `point` | Randomized voting public key |
| `van_new` | `fe` | New VAN commitment |
| `vc` | `fe` | Vote commitment |
| `rt_vct` | `fe` | VCT root at `anchor_height` |
| `anchor_height` | `u64` | Vote chain height of the anchor |
| `proposal_id` | `u8` | 1 to 50 |
| `ea_pk` | `point` | Election authority public key |

The rules for effectiveness are in [Vote Proof].

### Share Reveal Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 10 |
| `voting_round_id` | `bytes[32]` | |
| `proof` | `proof` | The Vote Reveal Proof $\pi_{\text{reveal}}$ |
| `share_nullifier` | `fe` | Share nullifier |
| `option_ciphertexts` | `ciphertext` $\times N_{\mathsf{opt}}$ | $E_0, \ldots, E_{N_{\mathsf{opt}} - 1}$ in position order |
| `proposal_id` | `u8` | 1 to 50 |
| `rt_vct` | `fe` | The round's final VCT root |

A share reveal transaction carries no signature and names no
submitter. The proof binds every field, and the verifier supplies the
round's $\mathsf{ea}\_\mathsf{pk}$ as the remaining public input. The
rules for effectiveness are in [Vote Reveal Proof].

### Partial Decryption Transaction

| Field | Type | Description |
|---|---|---|
| `tx_type` | `u8` | 11 |
| `voting_round_id` | `bytes[32]` | |
| `trustee` | `u8` | The trustee's index $i$ |
| `decryptions` | `vec<PartialDecryption>` | One entry per decrypted pair, in ascending order of proposal identifier and then option position |
| `signature` | `ed25519sig` | By trustee $i$'s account key |

A `PartialDecryption` is:

| Field | Type | Description |
|---|---|---|
| `D` | `point` | $D_i = [\mathsf{sk}_i]\, C_{1,\mathsf{agg}}$ |
| `dleq_e` | `scalar` | $e$ of the DLEQ proof |
| `dleq_z` | `scalar` | $z$ of the DLEQ proof |

The number of entries MUST equal the sum, over the round's proposals,
of each proposal's option count. The rules for effectiveness are in
[Partial Decryption].


## Poseidon Instantiation

All Poseidon hashes in this protocol use the same instantiation as
the nullifier non-membership tree defined in [^balance-proof]:
$\mathsf{P128Pow5T3}$ over $\mathbb{F}_{q_{\mathbb{P}}}$ (the Pallas
base field), with S-box $x^5$, width $t = 3$, rate $r = 2$, targeting
128-bit security, using the standard parameter generation procedure
from [^poseidon].

Hashes with $L$ inputs use $\mathsf{ConstantLength}\langle L \rangle$
mode (absorbing $L$ field elements with length padding), absorbing
two elements per permutation.

The following table lists every Poseidon call site in this protocol:

| Hash | Inputs ($L$) | Mode | Permutations |
|---|---|---|---|
| Voting round identifier | 13 | $\mathsf{ConstantLength}\langle 13 \rangle$ | 7 |
| VAN core | 6 | $\mathsf{ConstantLength}\langle 6 \rangle$ | 3 |
| VAN blinding | 2 | $\mathsf{ConstantLength}\langle 2 \rangle$ | 1 |
| VAN nullifier | 4 | $\mathsf{ConstantLength}\langle 4 \rangle$ | 2 |
| Vote commitment | 5 | $\mathsf{ConstantLength}\langle 5 \rangle$ | 3 |
| Blinded share commitment | 5 | $\mathsf{ConstantLength}\langle 5 \rangle$ | 3 |
| Shares hash | 16 | $\mathsf{ConstantLength}\langle 16 \rangle$ | 8 |
| Share nullifier | 4 | $\mathsf{ConstantLength}\langle 4 \rangle$ | 2 |
| VCT internal node | 2 | $\mathsf{ConstantLength}\langle 2 \rangle$ | 1 |
| Governance nullifier | 4 | $\mathsf{ConstantLength}\langle 4 \rangle$ | 2 |
| Rho binding | 7 | $\mathsf{ConstantLength}\langle 7 \rangle$ | 4 |

## Domain Separator Tags

Several constructions in this protocol use a domain separator tag
encoded as a Pallas scalar. Each tag is defined by a fixed ASCII
string. To convert a tag string to a field element, interpret its
byte representation as an unsigned little-endian integer:

$$\mathsf{tag} = \sum_{j=0}^{\ell-1} b_j \cdot 256^j$$

where $b_0, \ldots, b_{\ell-1}$ are the ASCII byte values of the string
and $\ell$ is its length. All tag strings in this protocol are shorter
than 32 bytes, so the resulting integer is less than $2^{256}$ and fits
in $\mathbb{F}_{q_{\mathbb{P}}}$ without reduction.

## Verification

This section defines what a party must check to establish that a
published result follows from the votes that were cast. It is stated
here because no other document in this set defines it, and because
verifying a subset of these steps establishes correspondingly less.
Nothing below is performed by the vote chain; a party that wants the
assurance performs it, and every party that does obtains the same
answer.

A verifier with a copy of the vote chain MUST check each of the
following. They are ordered so that each step presupposes the ones
above it.

1. **Round configuration.** The round's snapshot roots are correct, as
   established by the procedure in [Reading the Snapshot Roots]. This is
   not verifiable from the record, because the roots are supplied as
   input at round creation rather than derived from anything the record
   contains. A verifier that omits this step establishes only that
   votes are well formed *with respect to* roots it has not checked.
2. **Effectiveness.** Every recorded transaction of the round is
   classified as effective or not by the rules in
   [Effective Transactions], in record order: the round creation and
   any cancellation, the ceremony and its acknowledgements, and every
   delegation, vote and share reveal transaction, each with its proof
   verified and its out-of-circuit checks applied. This step derives
   the round's state, $\mathsf{ea}\_\mathsf{pk}$, the VCT and its final
   root, and the nullifier sets.
3. **Nullifier disjointness.** The three nullifier sets so derived
   contain no duplicates, so no voting authority was consumed twice.
   This follows from step 2 and is stated separately because a tool
   that reports it can be checked against one that does not.
4. **Aggregation.** For each $(\mathsf{proposal}\_\mathsf{id}, j)$ pair,
   the aggregate ciphertext is the component-wise sum of the
   position-$j$ ciphertexts of the round's effective share reveal
   transactions, every one of which is anchored to the final VCT root.
5. **Decryption.** The published per-option totals follow from the
   aggregates and from effective partial decryption transactions of at
   least $t$ trustees, each DLEQ proof verified against the verifier's
   own aggregate, by the combination in [Tally].
6. **Unit.** The published figures are interpreted in the unit in which
   they are denominated; see [Ballot Scaling] and [Tally units].

**What this establishes.** Steps 2 through 5 establish that the totals
are the correct sum of the votes present in the record. With step 1,
they establish that those votes were cast by holders of the balances
they claim.

**What it does not establish.** No step above, and no combination of
them, establishes that every vote cast was recorded. Exclusion of a
transaction leaves no evidence in the record; see
[Transaction Inclusion]. A verified result is therefore a lower bound
on the support each option received.

**On partial verification.** Checking step 5 alone confirms only that
the announced totals match aggregates the verifier did not derive — it
does not check any proof, and it does not check either snapshot root.
A tool or procedure that performs only the decryption check MUST NOT
be described as verifying a round's result. Implementations of
verification tooling SHOULD state which of the steps above they
perform.


# Rationale

## Why a Separate Vote Chain

Voting transactions (delegation proofs, vote proofs, share reveals) are
not standard Zcash shielded transactions. They require a new commitment
tree (the VCT), new nullifier sets, and a tally over recorded
ciphertexts with homomorphic aggregation. Recording them on a
purpose-built chain avoids modifying the Zcash consensus layer and
keeps the governance mechanism independent of mainchain upgrade
cycles. Because the chain records and does not validate (see
[Why the Vote Chain Validates Nothing]), what it needs from its
consensus is only ordering and availability.

## Why the Vote Chain Validates Nothing

Earlier revisions had the vote chain enforce this ZIP: verify each
proof at admission, reject a transaction reusing a nullifier, maintain
the VCT and running ciphertext sums, check the ceremony's proofs,
derive the election authority key, drive the round's state machine and
combine the partial decryptions into a tally. A verifier was still
expected to redo all of it, so validators were not trusted for the
result, but they were responsible for the rules, and that had three
costs.

First, a verifier could not distinguish a transaction the chain
rejected as invalid from one it declined to record: both were absent,
and "invalid" was available as an account of any exclusion. Second,
every rule was implemented twice, once in consensus and once in every
verifier, and the two could disagree, with consensus winning by
default. Third, the chain's own state — the accumulators, the recorded
final root, the derived key, the tally — was a second source of truth
that a wallet or an auditor could take instead of the record, and
doing so was the path of least resistance.

Recording without validating removes all three. The record is the only
artefact, every rule is a function of it, and every party that applies
the rules obtains the same derived state. Validators retain exactly
one capability, exclusion, and [Transaction Inclusion] describes it.
A transaction this ZIP treats as ineffective costs its submitter a fee
and nobody anything else.

The costs accepted are these. The record can hold transactions that
have no effect, bounded only by what the chain charges to record
them. A wallet needs the effective leaf set to build its Merkle path
and either derives it, which means verifying every proof in the
round, or takes the tree from a node it does not control, which can
cost it only its own submission if the node lies. And no party tells a
submitter that its transaction was ineffective; a wallet learns that
the way any verifier does, by applying the rules to what was
recorded.

## Why Deadlines Are Block Heights

A round's windows are stated in vote chain block heights rather than
in wall-clock time. Whether a transaction was recorded within a window
is then a property of the block it is in, which every reader of the
record sees identically, without reference to block timestamps that a
proposer influences or to any party's clock. A vote chain that stalls
extends the windows of every round on it rather than expiring them
while nothing can be recorded, which is the failure that would
otherwise lose votes. The submission schedule in [Submission Timing]
is drawn in blocks for the same reason ZIP 318's is: block height is
what an observer of the record measures. The cost is that wallets
present deadlines to voters as estimates, from the chain's published
block rate, and that the reveal window's duration in time varies with
that rate.
## Why Poseidon for the VCT

The Orchard note commitment tree uses Sinsemilla [^protocol-concretesinsemilla]
for Merkle hashing. The VCT uses Poseidon instead because all three ZKPs
in this protocol require VCT Merkle membership proofs, and Poseidon
operates natively on field elements, making it significantly more
efficient inside Halo 2 [^halo2] arithmetic circuits than Sinsemilla
(which is optimized for bitstring
inputs). Since the VCT is new infrastructure with no backwards-
compatibility constraint, the more circuit-efficient primitive is
appropriate. This is the same rationale as for the nullifier
non-membership tree in [^balance-proof].

## Why Ballot Scaling

Expressing vote values in ballots ($\lfloor \text{zatoshi} / 12{,}500{,}000 \rfloor$) rather
than raw zatoshi reduces bit-width throughout the protocol: El Gamal
scalar multiplications are faster, range checks are tighter, and the
discrete-log recovery at tally time (see [Decryption]) has a smaller
search space.
The 0.125 ZEC minimum also prevents dust delegations from bloating vote
chain state.

## Why a 24-bit Remainder Range

The remainder in the ballot scaling constraint is range-checked to
24 bits ($< 16{,}777{,}216$), which is wider than the divisor
($12{,}500{,}000$). A prover can set $r > 12{,}500{,}000$, effectively
shorting themselves one ballot. This is harmless:
$\mathsf{num}\_\mathsf{ballots}$ does not appear in any governance
nullifier, so the only effect is the prover voting with slightly less
weight than they could. The wider check avoids a custom
non-power-of-two range check in circuit.

## Why Delegation to a Hotkey

The protocol requires voters to produce Halo 2 zero-knowledge proofs
(delegation proof, vote proof) and construct vote commitments, operations that demand general-purpose computation on private key
material. Hardware wallets that custody Orchard spending keys cannot
perform these operations: they support signature generation but not
arbitrary-circuit ZKP construction.

Delegation to a governance hotkey resolves this by separating spend
authorization from vote execution. The hardware wallet signs a single
$\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$ during delegation, authorizing the
transfer of voting power to a software-controlled hotkey. All subsequent
voting operations (VAN consumption, vote commitment construction, share
submission) use the hotkey's key material and run on a general-purpose
device.

Without this separation, hardware wallet users would need to either
export spending keys to a software environment, negating the security
benefit of hardware custody, or forgo participation in governance
entirely. Delegation preserves the hardware wallet's role as the sole
custodian of spending keys while enabling full participation in the
voting protocol.

## Why Deterministic Hotkey Derivation

The voting flow spans multiple app sessions: delegation (ZKP #1),
one or more votes (ZKP #2), and vote signing. Each step requires
the hotkey's spend-authorizing key. The per-vote secrets ($r_i$,
$\mathsf{blind}\_\mathsf{i}$, $\alpha_v$) are also derived
deterministically from the hotkey seed via a domain-separated PRF
(see [Per-Vote Secret Derivation]), so crash recovery extends to
all secrets needed for vote construction.

Deriving every secret from a single seed means the wallet stores
only a BIP 39 mnemonic in its keychain. If the app is terminated
between delegation and voting, or between proposals, all key
material is reconstructed from the mnemonic. Randomly sampled keys
would require securely persisting each component ($\mathsf{vsk}$,
$\mathsf{vsk.nk}$, $\mathsf{rivk}\_\mathsf{v}$, and all per-vote
$r_i$ and $\mathsf{blind}\_\mathsf{i}$ values) independently,
and any storage failure would be unrecoverable.

## Why VAN Nullifier Domain Separation

The VAN nullifier uses $\mathsf{vsk.nk}$ (the governance hotkey's
nullifier deriving key) as the Poseidon key, while the governance
nullifier uses $\mathsf{nk}$ (the holder's nullifier deriving key).
When the hotkey is a separate key from the holder's, these are distinct
field elements, and cross-circuit collision resistance follows from the
key difference alone. When the hotkey reuses the holder's key hierarchy
(e.g., in an all-software flow without hardware wallet separation),
$\mathsf{vsk.nk} = \mathsf{nk}$ and collision resistance relies on
the domain tags being distinct: `"vote authority spend"` and
`"governance authorization"` differ in both byte length and content,
producing distinct field elements. The domain tags provide
defense-in-depth in both cases.

## Why 5 Notes per Delegation

The Delegation Proof fixes the note slot count at 5 (with padding for
holders who have fewer notes). This choice balances wallet coverage
against proof cost: empirical analysis of the shielded pool
shows that over 90% of wallets hold 5 or fewer notes, so most holders
can delegate their full balance with a single user-facing signature.

Each additional note slot adds a full set of per-note constraints to
the circuit, increasing proving time. A higher
slot count would cover marginally more wallets at a disproportionate
cost in prover resources. Holders with more than 5 notes can perform multiple delegations, each covering up to 5 notes.

## Why a Dummy Signed Note

The Delegation Proof includes a dummy signed note (value 0) whose rho is
deterministically bound to the delegation context. This mechanism exists
to obtain a $\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$ from hardware
wallets that support only the standard Orchard PCZT signing flow,
without requiring firmware changes specific to governance.

The dummy signed note construction, PCZT signing flow, and design
rationale (1-zatoshi display value, rho binding for non-replayability,
ZIP 244 sighash reuse, and the trade-offs of a future custom signing
protocol) are specified in [^balance-proof].

## Why $N_s$ Shares Per Vote

Splitting a vote into $N_s$ shares serves two purposes. First, it
bounds what a party able to decrypt a single ciphertext learns: one
share rather than the voter's whole ballot count. Second, it makes the
voter's on-chain footprint $N_s$ unlinkable reveals rather than one,
so that recovering the total requires grouping them.

Both purposes depend on the shares being ungroupable, and the protocol
is designed so that nothing it publishes groups them. Each reveal
carries a nullifier, ciphertexts and constants that are the same for
every reveal in the round; see [Privacy Implications]. Earlier drafts
undermined this by having a submission server construct the Vote
Reveal Proof, which required the server to be told which vote
commitment each share belonged to, so that a server holding two shares
of a vote could group them from the payload alone. That path is
removed; see [Why the Client Constructs Reveal Proofs].

With content linkage removed, the remaining channel is metadata:
submission height and network origin. Those are addressed by the
memoryless schedule in [Submission Timing] and the per-share network
isolation rules in [Share Submission], and by there being no party
between the client and the chain (see [Why There Are No Relays]).

Limiting what one decryption reveals depends on the decomposition
strategy. Under an even split, one decrypted share determines the total
to within $N_s$ ballots, so the EA's view of one share is equivalent
to its view of the whole ballot count. The requirement in [Vote Share]
exists to prevent this (see [Why Randomized Share Decomposition]).

Earlier drafts specified a fallback in which a voter casting near the
end of the voting window placed their full ballot count into a single
share. That mode is removed; see [Why There Is No Single-Share Mode].

## Why Randomized Share Decomposition

[Vote Share] requires that share values not be a deterministic function
of $\mathsf{num}\_\mathsf{ballots}$. The reason is that additive
splitting leaks the total to anyone who can decrypt a part of it, and
the size of that leak is set entirely by how the split is chosen.

Under an even split — floor division with the remainder placed in the
last share — every share except the last equals
$\lfloor \mathsf{num}\_\mathsf{ballots} / N_s \rfloor$. Decrypting any
one of them determines the ballot count to within $N_s$ ballots, which
at $N_s = 16$ is 2 ZEC. Under such a scheme vote splitting provides
essentially no balance hiding against a party that can decrypt a single
share, and the protection claimed in [Privacy Implications] would rest
on the threshold assumption alone.

The default decomposition in [Vote Share] samples a uniformly random
composition of $\mathsf{num}\_\mathsf{ballots}$ into $N_s$ parts. A
single share is then distributed over a wide range rather than
concentrated at the mean, so decrypting one share bounds the total far
more loosely.

This is a bound on what one decryption reveals, not a guarantee about
several. Any additive decomposition into a fixed number of parts leaks
information about the sum to a party holding more than one part: the
expected value of a share is $\mathsf{num}\_\mathsf{ballots} / N_s$
under any scheme, so an adversary that can group several shares of one
vote and decrypt them recovers the total with accuracy improving in the
number it holds, and exactly with all $N_s$. Randomization is therefore
paired with, and not a substitute for, the unlinkability of reveals
established in [Privacy Implications] and the submission discipline in
[Share Submission]: the decomposition limits what a single decryption
reveals, and unlinkability denies the adversary the grouping that would
let it combine several.

## Why Blinded Share Commitments

Each share commitment includes a random blind factor:
$\mathsf{share}\_{\mathsf{comm}\_\mathsf{i}} = \mathsf{Poseidon}(\mathsf{blind}\_\mathsf{i}, C_{1,i,x}, C_{2,i,x}, C_{1,i,y}, C_{2,i,y})$.
Without blinding, an observer could compute
$\mathsf{Poseidon}(C_{1,i,x}, C_{2,i,x}, C_{1,i,y}, C_{2,i,y})$ for each on-chain ciphertext
and compare against the $\mathsf{shares}\_\mathsf{hash}$ values committed in
VCs, linking revealed shares back to specific vote commitments. The
blind factor makes this reverse computation infeasible.

## Why Share Commitments Bind Full Curve Points

On the Pallas curve, every $x$-coordinate has two valid $y$-values:
$P$ and $-P$. If the share commitment bound only the $x$-coordinates
of the ciphertext points, a voter could open a commitment to the
negation of the ciphertext it committed: $-C_1$ and $-C_2$ have the
same $x$-coordinates as $C_1$ and $C_2$, so condition 4 of the Vote
Reveal Proof would be satisfied by either. The negated ciphertext
encrypts $-v$ instead of $v$ under the same El Gamal key. The Vote
Proof's range check on $v$ (condition 9) would be bypassed at reveal,
the tally would accumulate $\mathsf{Enc}(-v)$, and a voter could
subtract weight from an option rather than add it.

Including both $x$- and $y$-coordinates in the share commitment hash
binds each commitment to the exact curve point. Opening to the
negation changes the $y$-coordinate, producing a different
$\mathsf{share}\_\mathsf{comm}$, which cascades through
$\mathsf{shares}\_\mathsf{hash} \to \mathsf{vc} \to$ Merkle root,
invalidating the proof. The public option-vector coordinates are
likewise bound in full by the Vote Reveal Proof, so a block proposer
cannot negate a revealed ciphertext after the fact either.

Full $y$-coordinates are used rather than 1-bit sign values because
the $y$-cells are already available from the ECC gadget output in the
Vote Proof circuit; extracting parity bits in-circuit would require a
255-bit field decomposition gadget, adding substantial constraint cost
for no security benefit.

## Why the Client Constructs Reveal Proofs

Earlier revisions of this ZIP had a submission server construct the
Vote Reveal Proof on the voter's behalf, on two grounds: that mobile
devices are unreliable for background ZKP computation, and that
server-side construction enables temporal mixing of shares from many
voters. Neither ground survives examination, and the design had a cost
that no encoding of the payload could remove.

A prover must be told which leaf of the VCT it is proving membership
of. Every field a server needed — the vote commitment, its VCT
position, the shares hash, the array of share commitments — takes the
same value in all $N_s$ payloads of one vote, so a server receiving two
of a voter's payloads could group them with certainty, from the
contents alone, before any timing or network measure applied. The
correlation was inherent in delegating proof construction: any leaf
identifier links the $N_s$ shares, because they all descend from one
vote transaction. Splitting the vote into $N_s$ separately inserted
leaves would not have helped, since the leaves would be inserted
consecutively by that transaction, and inserting them separately
would require a linking proof per share that only the client could
construct.

The cost of constructing the proof on the client, by contrast, is
small. Every client that votes already constructs the Vote Proof,
which contains $N_s$ El Gamal encryptions and a VAN spend; the Vote
Reveal Proof is smaller than that, and a client constructs it once per
share. The class of clients that can vote but cannot construct a Vote
Reveal Proof is empty.

Content linkage had to be removed before timing and network measures
had anything to protect: a server told which shares belong together
does not need to infer it. With the client holding everything and the
payload reduced to the message itself, those measures are effective,
and [Share Submission] requires them.

## Why There Are No Relays

After proof construction moved to the client, an intermediate revision
kept a store-and-forward relay: a client that would not be online
across its submission schedule could hand each finished message to a
relay with the time at which to submit it. The relay received nothing
a chain observer would not, and rules limited it to one message per
vote. It is removed for two reasons.

The first is that delayed submission is the client's duty. The
schedule in [Submission Timing] protects the voter only if the client
carries it out, and a party that submits on the client's behalf holds,
for each message it is given, the client's network origin and the
requested time, which are the two quantities the schedule exists to
decorrelate. ZIP 318 faces the same problem for pool-crossing
transfers and resolves it the same way: the wallet comes online to
submit, in background sessions where the platform allows and on the
next application open where it does not, one overdue transaction at a
time. This ZIP adopts those rules.

The second is that the one-message-per-relay rule that was meant to
contain a relay's view was itself a leak. It made the assignment of a
vote's messages to relays a random injection rather than independent
draws, so any two messages at one relay were known to belong to
different votes, and a coalition observing several relays could use
that to prune the pairings it had to consider. A rule that exists to
protect against a relay that has received two shares of one vote,
which the network isolation rules already forbid the client to allow,
was paying for that protection with information given to every relay.

With relays gone, the only party that ever holds a share reveal
message before it is recorded is the client that built it, and the
only metadata any party sees is what the vote chain node receiving a
single submission sees. Availability is the client's problem, as it is
for every other transaction a wallet sends, and the reveal window is
sized so that ordinary wallet use covers it (see
[Why a Separate Reveal Window]).

## Why There Is No Single-Share Mode

Placing a voter's entire ballot count into a single share removes every
protection this ZIP provides for vote amounts at once. One decryption
recovers the exact figure — not an estimate bounded by a decomposition
strategy, as in [Why Randomized Share Decomposition], but the value
itself.

The justification for accepting this was that a submission server might
not complete $N_s$ Vote Reveal Proofs before the voting window closed,
so a voter casting late would otherwise lose their vote. That
constraint no longer exists: the client constructs its own proofs,
does so during a reveal window that opens only after voting has
closed, and can construct all $N_s$ of them in seconds at any point in
that window. A voter who is late constructs and submits under the
compressed schedule in [Submission Timing], which preserves both
inclusion and amount privacy.

Implementations should note that this mode's exposure was
disproportionately borne by voters who waited — including those waiting
deliberately to avoid influencing others — and that its on-chain
indistinguishability, which earlier drafts cited, protected against a
chain observer while the submission server of the time could identify
such a vote directly from the payload.

## Why Decisions Are Encrypted at Reveal

Earlier revisions of this ZIP made $\mathsf{vote}\_\mathsf{decision}$ a
public input to the Vote Reveal Proof, so that the chain could route
each revealed ciphertext to the accumulator for that option. Every
submission server and every chain observer learned the decision
attached to each revealed share, running per-option totals were public
while a round was open, and validators could exclude reveals by the
option they supported without decrypting anything. Those revisions
declined to encrypt decisions on cost grounds: a per-option ciphertext
vector, computed in the Vote Proof, would multiply that circuit's
sixteen encryptions by the number of options.

The cost is avoidable by producing the vector at reveal rather than at
vote. The Vote Commitment continues to bind one ciphertext per share
and the decision as a private witness. The Vote Reveal Proof then
publishes $N_{\mathsf{opt}}$ ciphertexts, places the committed
ciphertext at the decision's position, fills every other position with
a fresh encryption of zero, and proves both facts. The Vote Proof is
unchanged. The reveal circuit gains $N_{\mathsf{opt}}$ fixed-base and
$N_{\mathsf{opt}}$ variable-base scalar multiplications, once per
share, which is well under the cost of the Vote Proof the client has
already constructed.

The padding must use independent randomness per position. With one
scalar $\rho$ across the vector, the second components of two
positions would differ by $[v]\, G$ where $v$ is the share value, and
$v < 2^{30}$ is recoverable by a bounded discrete logarithm without
any key. Condition 7 of [Vote Reveal Proof] therefore requires a
distinct $\rho_j$ per position, and the circuit computes a padding
ciphertext for every position, including the one it discards, so that
the circuit's shape does not depend on the decision.

The committed ciphertext is placed in the vector as it was committed,
without re-randomisation. It appears on chain exactly once, in the
reveal that opens it, so there is no second appearance to link it to.

Proposal identifiers remain public. The vote transaction already
exposes $\mathsf{proposal}\_\mathsf{id}$ as a public input of the Vote
Proof, so concealing it at reveal would conceal nothing, and it would
require the vector to span every option of every proposal in the
round. A deployment that wants a round's proposals to be
indistinguishable runs one proposal per round.

The consequence is that this protocol provides ballot secrecy against
validators, vote chain nodes and chain observers, and against any party
holding fewer than $t$ key shares. A coalition of $t$ trustees that decrypts an
individual share learns that share's option along with its value,
within the limits stated in [Privacy Implications].

## Why Not TEE-Based Proof Construction

Running Vote Reveal Proof construction inside a Trusted Execution
Environment was considered as a way to let a server construct proofs
without observing the witness material or the decision. TEEs introduce
infrastructure complexity, rely on vendor-specific trust assumptions,
and are subject to side-channel attacks demonstrated against SGX and
comparable platforms.

With the client constructing its own proofs (see [Share Submission]),
the problem a TEE would solve does not arise: no server receives
anything to protect, and no hardware assumption is required of the
operator set.

## Why ZIP 318 Scheduling

The submission schedule in [Submission Timing] is taken from ZIP 318
[^zip-0318] rather than designed independently, because the problem is
the one ZIP 318 already solved.

In a pool migration, a wallet emits several pool-crossing transfers
whose amounts are individually visible and whose sum is the quantity to
be protected. In this protocol, a voter emits $N_s$ shares whose sum is
the ballot count. In both cases an adversary that can group one
client's emissions recovers the total, and in both cases the defence
has three parts: the values must not be a deterministic function of the
total, the order in which they are emitted must not depend on their
magnitudes, and the times at which they are emitted must be memoryless
so that a burst does not identify a single client's set.

ZIP 318 specifies all three. Earlier revisions of this protocol
specified only that shares be submitted at "randomized delays", without
a distribution, a spread, or an ordering requirement, and specified an
even decomposition that made the first part vacuous. That is the weaker
form of the same design, arrived at independently, and the difference
was not visible while the two documents were read separately.

Adopting ZIP 318's discipline also has a review benefit: the analysis
supporting it — in particular why memoryless inter-arrival delays are
preferable to fixed or minimum-separated ones — has already been
reviewed in that context and need not be re-derived here.

ZIP 318's rules for a wallet that is not running when a transaction
falls due — best-effort background sessions that broadcast without
synchronising, reconciliation on every application open, and at most
one overdue transaction per open — are adopted as well, because a
wallet that cannot be relied on to be online is the case that
motivated relays, and these rules are how ZIP 318 handles it without
any third party (see [Why There Are No Relays]).

Two differences from ZIP 318 are deliberate. First, delays here are
expressed in vote chain block deltas against the round's
$\mathsf{reveal}\_\mathsf{end}\_\mathsf{height}$ rather than in
Zcash block deltas, because the deadline is a vote chain parameter and
the vote chain's block rate is not the Zcash block rate. Second,
$\mathsf{MEAN}\_\mathsf{DELAY}$ is derived from the remaining window
rather than fixed, because a reveal window's duration is a per-round
configuration value while a migration's duration is chosen by the
wallet.

## Why Reusing VAN Address and Randomness

When a vote consumes a VAN and produces a new one, the new VAN reuses
the old VAN's diversified address ($\mathsf{vpk}\_{\mathsf{g}\_\mathsf{d}}$,
$\mathsf{vpk}\_{\mathsf{pk}\_\mathsf{d}}$) and commitment randomness
($\mathsf{gov}\_{\mathsf{comm}\_\mathsf{rand}}$). Only
$\mathsf{proposal}\_\mathsf{authority}$ changes. This is safe because VAN
commitments are blinded Poseidon hashes - the shared fields are never
externally observable (both old and new commitments appear as opaque
field elements in the VCT), so address rotation would provide no
additional unlinkability.

## Why Proposal Identifiers Start at 1

The $\mathsf{proposal}\_\mathsf{authority}$ bitmask is 51 bits wide, but
$\mathsf{proposal}\_\mathsf{id}$ values start at 1, yielding 50 usable
proposal slots rather than 51. Bit 0 is reserved.

This follows from how lookup arguments work in Halo 2. A lookup table
that validates $(\mathsf{proposal}\_\mathsf{id}, 2^{\mathsf{proposal}\_\mathsf{id}})$
must include a default row that satisfies the lookup when the selector
is inactive. The circuit uses the identity row $(0, 1)$ for this
purpose: when the selector $q = 0$, the lookup input evaluates to
$(0, 1)$, which must be present in the table for the proof to verify.
Because every inactive row matches this entry, $\mathsf{proposal}\_\mathsf{id} = 0$
cannot be used as a valid proposal identifier: a prover could trivially
satisfy the lookup with $\mathsf{proposal}\_\mathsf{id} = 0$ even when no
authority check is intended. An additional non-zero gate
($\mathsf{proposal}\_\mathsf{id} \cdot \mathsf{proposal}\_\mathsf{id}^{-1} = 1$)
provides defense-in-depth by rejecting $\mathsf{proposal}\_\mathsf{id} = 0$
on active rows.

## Why Distributed Key Generation

The election authority key is generated in shares from the start, so
that compromise of any set of trustees smaller than $t$ does not expose
$\mathsf{ea}\_\mathsf{sk}$ and therefore cannot open an individual
share ciphertext, and so that no party ever holds the key at all.

Earlier revisions used a trusted dealer: one party sampled the key,
split it with Shamir's scheme, distributed the shares and was trusted
to erase its copy. Erasure is not verifiable by any other party, so
every amount-privacy claim rested on one party's conduct during a
window nobody else could observe, and a deployment could at best name
the party. Distributed key generation removes the window rather than
naming the party who had it.

The construction is Pedersen's, with each participant proving
knowledge of its constant term before any other participant's
commitment is fixed. Without that proof a participant that publishes
last can choose its commitment as a function of the others', which
lets it bias the resulting public key or cancel other contributions
entirely [^gjkr]. For an encryption key the bias itself would not help
an adversary decrypt, but the cancellation would let a single
participant substitute a key it controls, and the proof of knowledge
closes both at the cost of one Schnorr proof per participant. The vote
chain supplies the authenticated, ordered broadcast channel the
protocol assumes.

The output is a set of Shamir shares of a key that exists only as a
sum, and every downstream step — trustee share keys, partial
decryptions, DLEQ proofs and Lagrange combination — is unchanged from
the dealer-based design.

## Why a Send-Based VAN Model

The Vote Proof consumes the old VAN and produces a new one with the
voted proposal's authority bit cleared. This is a UTXO-style
"send": each vote appends two leaves to the VCT (new VAN + VC) and
requires the voter to hold a current Merkle path.

An alternative design was considered in which the VAN is eliminated
entirely and delegation is performed by out-of-band sharing of a
private key (hash-committed to on-chain). Under this model the Vote
Proof would not consume or re-create a VAN, removing the proposal
authority decrement step and the corresponding VCT growth. The primary
appeal is reduced client sync overhead: voters would not need to track
VAN re-insertions or maintain up-to-date Merkle paths for their own
VANs.

The send-based model is retained because it preserves the ability to
add partial delegation in a future extension, splitting
$\mathsf{num}\_\mathsf{ballots}$ across multiple delegates, each
receiving a fraction of the holder's voting weight. The VAN's explicit
ballot count and proposal authority bitmask are the data model that
would enable this; a future VAN-to-VAN delegation proof could consume
one VAN and produce two with subdivided ballot counts. Partial
delegation is not specified in this ZIP but the VAN model keeps the
design space open. Under the keysharing alternative, the delegate holds
the full key and therefore the full voting weight; there is no
in-protocol mechanism to subdivide it, and adding one later would
require a fundamentally different data model.

The sync savings of the alternative are also smaller than they first
appear: even without VAN re-creation, clients still need to update
VCT Merkle paths for their Vote Commitments (which the Vote Reveal
Proof requires). The VAN model adds
incremental path-update overhead but does not introduce a new
category of sync obligation. If a future design change eliminated the
need for clients to track VCT paths entirely (for example by private
retrieval of Merkle paths from a server), the tradeoff
would shift in favor of removing the VAN. See [Open issues].

## Why Classical El Gamal Rather Than Post-Quantum Encryption

El Gamal on Pallas is used because it is additively homomorphic, which
[Tally] requires, and because it reuses a curve already present in
Orchard. It is not post-quantum: an adversary running Shor's algorithm
could recover $\mathsf{ea}\_\mathsf{sk}$ from
$\mathsf{ea}\_\mathsf{pk}$ and decrypt individual share ciphertexts for
any round whose ciphertexts were recorded. No aggregatable
post-quantum encryption scheme suitable for this construction is
available at the time of writing. The key consequence for this
protocol is that vote-amount privacy has a finite horizon tied to
quantum computing timelines, while voter *identity* is unaffected
(alternate nullifier unlinkability relies on Poseidon preimage
resistance, not on El Gamal). Vote splitting across $N_s$ shares
provides additional mitigation: a quantum adversary would recover
individual shares rather than complete ballot allocations unless it
also breaks the Poseidon-based blinded share commitments.

## Why VCT Depth 24

The Orchard note commitment tree uses depth 32, supporting $2^{32}$
(~4.3 billion) leaves. Governance voting produces far fewer leaves:
each voter generates one VAN per delegation and two leaves (a new VAN
plus a VC) per vote. Even 10,000 voters each voting on 50 proposals
produce roughly 1 million leaves, well within the $2^{24}$
(~16.7 million) capacity of a depth-24 tree. Because each voting round
maintains its own VCT, this budget applies per round rather than
accumulating across rounds.

The reduced depth saves constraint rows in every circuit that performs
a Merkle membership proof (Vote Proof and Vote Reveal Proof), since
each level adds a Poseidon hash region. Moving from 32 to 24 levels
removes 8 hash regions per proof, reducing prover cost and verification
time without any practical capacity risk.


## Why Domain Tags in the VCT

Both VANs and VCs are leaves in the same Merkle tree. The domain tags
($\mathsf{DOMAIN}\_\mathsf{VAN} = 0$, $\mathsf{DOMAIN}\_\mathsf{VC} = 1$) as the first
Poseidon input make it structurally impossible for a valid VAN preimage
to produce the same hash as a valid VC preimage, regardless of the
remaining inputs.


## Why Bind to a Block Hash

Anchoring a round to a height alone leaves its meaning dependent on
which chain the reader follows. Binding it to
$(H, \mathsf{snapshot}\_\mathsf{blockhash})$ makes a reorganisation
affecting the snapshot a detectable condition with a specified response,
rather than a silent change in the eligible note set.

## Why Ratification Precedes Voting

A round names $\mathsf{ea}\_\mathsf{pk}$, but a public key is not a
statement by anybody that they will use the corresponding shares. A
voter who delegates and votes does work, and does it in the expectation
that the round will produce a result. A statement collected after the
round records what happened; one collected before it opens is an input
the voter can act on. Ratification also carries the trustees'
independent check of the snapshot roots, so a round that opens has had
its snapshot verified by every party that can decrypt its result, not
only by the party that proposed it.

Ratification is unanimous for the reason stated in [Ratification]: the
threshold $t$ is the number of trustees needed at tally time, and a
round that opens with exactly that many committed has no margin for the
loss of any of them.

Unanimity gives every trustee a veto over opening, exercised by doing
nothing. [Cancellation] gives the same parties, and the creator, a way
to exercise it explicitly and at once, so that a round stalled by one
trustee can be replaced without waiting for the ceremony grid to run
out and without the replaced round later opening on a late
acknowledgement.

## Why a Separate Reveal Window

Every Vote Reveal Proof anchors to a VCT root, and the anchor is
public. Earlier revisions accepted any published root and let voting
and reveal overlap, so a voter constructing $N_s$ proofs against the
root current at the time would tag all $N_s$ reveals with a value few
other voters shared, and a chain observer could group them by it
without decrypting anything. That is the linkage this ZIP's privacy
claims cannot survive.

Two remedies were considered. Periodic checkpoint roots, with reveals
required to anchor to a checkpoint, bound the tag's resolution to the
checkpoint interval but still partition voters into cohorts by the
interval their vote landed in, and early voters in a thin round form
cohorts of one. Closing the voting window before any reveal is accepted
removes the partition entirely: the VCT is frozen, there is exactly one
root, and every reveal in the round carries it.

The cost is that a wallet cannot construct its reveal proofs at the
moment it votes, because the final root does not exist yet. It must
come online during the reveal window, construct the proofs, and submit
them across the window at the heights it draws, in background sessions
or on later opens (see [Submission Timing]). A wallet that never
returns during the reveal window loses its vote, and one that returns
only once reveals only part of it. That cost is bounded by the length
of the reveal window, which is why [Round Lifecycle] requires a
deployment to publish it and choose it with ordinary wallet usage in
mind.

A secondary benefit is that the client's Merkle path is final once the
round enters REVEALING. A wallet syncs the round's tree once, after
voting closes, rather than maintaining a witness across the voting
window.

# Deployment

This ZIP does not specify a consensus change to the Zcash mainchain.

The parameters below are specific to a deployment rather than to the
protocol, but they are recorded here, with the protocol they
parameterise, rather than in a separate operational document. A
parameter stated apart from the claim that depends on it can drift from
it without either document becoming self-inconsistent.

A deployment MUST publish the values it uses for each parameter in this
section.

## Protocol parameters

| Parameter | Value | Constraint |
|---|---|---|
| $N_s$ | 16 | Shares per vote commitment; fixed by the circuits. See [Vote Share]. |
| $N_{\mathsf{opt}}$ | 8 | Option positions per share reveal; the maximum options per proposal. See [Vote Reveal Proof]. |
| $\mathsf{MAX}\_\mathsf{PROPOSALS}$ | 50 | Proposals per round; fixed by the 51-bit authority bitmask. See [Proposals and Decisions]. |
| Ballot unit | 12,500,000 zatoshi | 0.125 ZEC per ballot; see [Ballot Scaling]. |
| Share range | $[0, 2^{30})$ | Per-share plaintext bound. |
| Decomposition | Randomized | MUST satisfy [Vote Share]; even splitting is forbidden. |
| $\Delta$ | One hour of blocks at the published block rate | Safety margin before $\mathsf{reveal}\_\mathsf{end}\_\mathsf{height}$, in blocks; see [Submission Timing]. |
| $\mathsf{MAX}\_\mathsf{DELAY}$ | $W / 4$ | Delay draws above this are discarded and redrawn. |

## Round parameters

| Parameter | Why it is published |
|---|---|
| The vote chain's block rate | Converts the heights in this ZIP to time for voters and for the submission schedule; see [Submission Timing]. |
| The decryption threshold $t$ and trustee count $n$ | Bounds every amount-privacy claim in the protocol; see [Election Authority Key Ceremony]. |
| The organisation acting as each trustee, its account key and its ceremony key | Allows the role separation required in [Requirements] to be checked, lets wallets verify acknowledgements, and lets any party recompute the trustee share keys. |
| `stage_window` and `max_attempts` | Round creation fields that set the ceremony grid; see [Election Authority Key Ceremony]. |
| The reveal window length, $\mathsf{reveal}\_\mathsf{end}\_\mathsf{height} - \mathsf{vote}\_\mathsf{end}\_\mathsf{height}$ | Bounds the period in which a wallet must return to reveal; see [Round Lifecycle]. |
| The poll runner and its signing key | Establishes whose poll signature wallets recognise; see [Poll Signature]. |
| $\mathsf{min}\_\mathsf{confirmations}$ | The confirmation depth used when choosing the snapshot; see [Snapshot Configuration]. |
| The trustee share retention period | Bounds the period over which amount-privacy claims hold; see [Election Authority Key Custody]. |
| The validator stake distribution | Sizes the coalition able to exclude transactions; see [Transaction Inclusion]. |

The RECOMMENDED value of $\mathsf{min}\_\mathsf{confirmations}$ is 100
blocks.

## Tally units

Tallies are denominated in ballots, not ZEC. A deployment MUST state
the unit of any published threshold, quorum or result. A threshold
expressed as 1,000,000 ZEC is 8,000,000 ballots.

## Implementation versions

Because the circuits, the vote chain and the client library evolve
independently, a deployment MUST publish the exact versions in use for
a round, and any party reproducing or auditing a round MUST pin them.
Recording the version of the client library alone is insufficient: the
circuits determine what the proofs mean.

# Reference implementation

- [^ref-circuits] — Halo 2 circuits for the Delegation Proof, Vote
  Proof, and Vote Reveal Proof.
- [^ref-vote-sdk] — Cosmos SDK vote chain. It validates transactions,
  maintains round state and tallies in consensus, and includes a
  submission server, none of which this ZIP specifies; see
  [Why the Vote Chain Validates Nothing] and [Why There Are No Relays].
- [^ref-nullifier-pir] — PIR server and client for privately retrieving
  nullifier non-membership proofs.
- [^ref-librustvoting] — Client-side Rust library for proof generation,
  vote construction, tree synchronization, and governance PCZT
  construction.


# Open issues

- A future custom signing protocol purpose-built for proof-of-balance
  could simplify the Delegation Proof circuit by removing the dummy
  signed note scaffolding and enable further improvements described in
  [^balance-proof].
- Partial delegation (a VAN-to-VAN delegation proof that consumes
  one VAN and produces two with subdivided $\mathsf{num}\_\mathsf{ballots}$)
  is enabled by the send-based VAN model but not specified in this
  ZIP. Specifying the circuit and transaction type would allow a
  holder to distribute voting weight across multiple delegates.
  See [Why a Send-Based VAN Model].
- A simplified non-send VAN model, replacing VAN consumption and
  re-creation with out-of-band key delegation, would remove the
  proposal authority decrement step from the Vote Proof and reduce
  per-vote VCT growth. This is currently not adopted because it
  forecloses partial delegation and clients still need VCT Merkle path
  updates regardless. If future design changes remove the client's need
  to track VCT paths (e.g., full server-side path retrieval), this
  tradeoff should be revisited.
  See [Why a Send-Based VAN Model].
- A coalition holding $t$ key shares can decrypt any individual share
  ciphertext. The protocol relies on the unlinkability of reveals (see
  [Privacy Implications]) to keep such a coalition from recovering any
  voter's total; an encryption layer that admitted opening of
  aggregates only would remove even the per-share exposure, and is not
  specified here.
- Encrypted shares are recorded on the vote chain permanently, and the
  El Gamal layer is not post-quantum. The plaintext is a fragment of a
  voter's shielded balance at the snapshot, which does not become less
  sensitive with time. [Non-requirements] places post-quantum security
  out of scope; that exclusion should be revisited, because the
  combination of a permanent public record and a non-post-quantum
  encryption layer means the exposure above has no expiry.
- Voters have no privacy-preserving way to confirm that their shares
  were included on the vote chain. A voter can observe the chain, but
  querying it for their own share nullifiers reveals which nullifiers
  are theirs. A private-retrieval confirmation mechanism would close
  this; adapting one for share nullifier
  queries requires additional specification. Until then, a voter cannot
  verify their own vote was counted, and omission of a share is not
  detectable by the voter who cast it.
- A voter who is the sole participant on an unpopular option may have
  their exact balance revealed by the tally, since the aggregate for
  that option equals their individual contribution. An opt-in mechanism
  to amend the declared ballot count — lowering, rounding or padding it,
  and proving the amendment in zero knowledge — would mitigate this.
- **Metadata at the point of submission.** [Share Submission] relies on
  the client to submit each message over its own network path. A vote
  chain node that receives a submission sees its origin, and a
  coalition of nodes pooling arrival logs with a coalition of $t$
  trustees could correlate by metadata what the protocol does not
  correlate by content if clients do not honour that rule. Nothing in
  the protocol can check that they do.
- **Partial reveal.** A wallet that returns during the reveal window
  only once, on a platform that grants no background execution, submits
  one overdue message per open under [Submission Timing] and may reveal
  only part of its weight. A verifier cannot distinguish that from a
  smaller vote. The reveal window length is the only lever.
- **Nullifier index maintenance.** A Zcash consensus node that serves
  $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ must maintain the
  index across chain reorganisations — tracking the block hash at which
  each nullifier entered, rolling back to the fork point before
  applying a new best chain, and refusing to serve a root for a block
  it has not confirmed is on its best chain. That guidance belongs with
  the tree's specification in [^balance-proof] and is not yet there.
- Open issues related to the balance proof are tracked in [^balance-proof].


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^protocol-concretespendauthsig]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1]. Section 5.4.7.1: Spend Authorization Signature (Orchard)](protocol/protocol.pdf#concretespendauthsig)

[^protocol-orchardkeycomponents]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1]. Section 4.2.3: Orchard Key Components](protocol/protocol.pdf#orchardkeycomponents)

[^bip39]: [M. Palatinus, P. Rusnak, A. Voisine, and S. Bowe, "BIP 39: Mnemonic code for generating deterministic keys", 2013](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)

[^shamir]: [A. Shamir, "How to share a secret", Communications of the ACM, vol. 22, no. 11, pp. 612-613, 1979](https://doi.org/10.1145/359168.359176)

[^pedersen-dkg]: [T. P. Pedersen, "A Threshold Cryptosystem without a Trusted Party", EUROCRYPT 1991](https://link.springer.com/chapter/10.1007/3-540-46416-6_47)

[^feldman]: [P. Feldman, "A Practical Scheme for Non-interactive Verifiable Secret Sharing", FOCS 1987](https://doi.org/10.1109/SFCS.1987.4)

[^gjkr]: [R. Gennaro, S. Jarecki, H. Krawczyk, and T. Rabin, "Secure Distributed Key Generation for Discrete-Log Based Cryptosystems", Journal of Cryptology 20(1), 2007](https://doi.org/10.1007/s00145-006-0347-3)

[^frost]: [C. Komlo and I. Goldberg, "FROST: Flexible Round-Optimized Schnorr Threshold Signatures", SAC 2020](https://eprint.iacr.org/2020/852)

[^chaum-pedersen]: [D. Chaum and T. P. Pedersen, "Wallet Databases with Observers", CRYPTO 1992](https://link.springer.com/chapter/10.1007/3-540-48071-4_7)

[^ecies]: [V. Shoup, "A Proposal for an ISO Standard for Public Key Encryption", version 2.1, 2001](https://www.shoup.net/papers/iso-2_1.pdf)

[^blake2]: [J.-P. Aumasson, S. Neves, Z. Wilcox-O'Hearn, and C. Winnerlein, "BLAKE2: simpler, smaller, fast as MD5", 2013](https://blake2.net/blake2.pdf)

[^poseidon]: [Poseidon: A New Hash Function for Zero-Knowledge Proof Systems](https://eprint.iacr.org/2019/458)

[^balance-proof]: [Orchard Proof-of-Balance](draft-valargroup-orchard-balance-proof.md)

[^zip-0318]: [ZIP 318: Orchard to Ironwood Migration](zip-0318.md)

[^zip-0225]: [ZIP 225: Version 5 Transaction Format](zip-0225.rst)

[^rfc8032]: [RFC 8032: Edwards-Curve Digital Signature Algorithm (EdDSA)](https://www.rfc-editor.org/rfc/rfc8032.html)

[^nullifier-pir]: [Draft ZIP: Nullifier Private Information Retrieval](draft-valargroup-nullifier-pir.md)

[^wallet-api]: [Draft ZIP: Shielded Voting Wallet API](draft-valargroup-shielded-voting-wallet-api.md)

[^voting-setup]: [Zcash Shielded Coinholder Voting](draft-valargroup-shielded-voting-setup.md)


[^halo2]: [S. Bowe, J. Grigg, and D. Hopwood, "Recursive Proof Composition without a Trusted Setup", 2019](https://eprint.iacr.org/2019/1021)

[^protocol-merkletree]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1]. Section 3.8: Note Commitment Trees](protocol/protocol.pdf#merkletree)

[^protocol-concretesinsemilla]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1]. Section 5.4.1.9: Sinsemilla Hash Function](protocol/protocol.pdf#concretesinsemillahash)

[^protocol-concretecommitivk]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1]. Section 5.4.9.4: CommitIvk](protocol/protocol.pdf#concretecommitivk)

[^protocol-pallasandvesta]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1]. Section 5.4.9.6: Pallas and Vesta](protocol/protocol.pdf#pallasandvesta)

[^zip-244]: [ZIP 244: Transaction Identifier and Signature Validation for v5 Transactions](zip-0244.rst)

[^pczt]: [zcash/zips issue #693: Standardize a protocol for creating shielded transactions offline (PCZT)](https://github.com/zcash/zips/issues/693)

[^ref-circuits]: [valargroup/voting-circuits: Halo 2 ZKP circuits for shielded voting](https://github.com/valargroup/voting-circuits)

[^ref-vote-sdk]: [valargroup/vote-sdk: Cosmos SDK vote chain for shielded voting](https://github.com/valargroup/vote-sdk)

[^ref-nullifier-pir]: [valargroup/vote-nullifier-pir: PIR system for nullifier non-membership proofs](https://github.com/valargroup/vote-nullifier-pir)

[^ref-librustvoting]: [valargroup/librustvoting: Client-side voting library for proof generation and vote construction](https://github.com/valargroup/librustvoting)
