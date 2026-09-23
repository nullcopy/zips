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

The terms below are to be interpreted as follows:

Administrator

: A party whose signature over a round's defining fields wallets
  recognise. See [Round Attestation].

Attestation

: An administrator's signature over a round's defining fields,
  establishing to wallets that the round's configuration is the one the
  administrator independently verified. See [Round Attestation].

Ballot

: A unit of voting weight derived from a zatoshi balance. The
  conversion from zatoshi to ballots is defined in [Ballot Scaling].

Decision

: A voter's chosen option for a proposal, represented as the option's
  0-indexed position in the proposal's option list. A decision is
  committed to in the Vote Commitment and is never published in
  cleartext; see [Proposals and Decisions] and [Vote Reveal Proof].

Decryption threshold ($t$)

: The number of key-share holders whose partial decryptions are needed
  to decrypt an aggregate ciphertext. Fixed by
  [Election Authority Key Ceremony].

Delegation

: The first phase of the protocol, in which a holder proves ownership
  of notes at the snapshot and transfers voting authority to a
  governance hotkey, producing a Vote Authority Note. See
  [Delegation Phase]. Distinct from the relaying of finished share
  reveal messages, which delegates nothing.

Election authority (EA)

: The El Gamal keypair under which a round's vote shares are encrypted,
  and whose private key decrypts the aggregate tally. A fresh keypair is
  generated for each round by distributed key generation among the
  round's key-share holders, so that no party ever holds the private
  key and decrypting the tally requires a threshold of holders acting
  together. See [Election Authority Key Ceremony] and
  [Election Authority Key Custody].

Election authority key ceremony

: The distributed key generation protocol, run on the vote chain, that
  produces a round's election authority public key and each holder's
  key share. See [Election Authority Key Ceremony].

Encrypted share accumulator

: Per-round vote chain state holding, for each proposal and each option
  position, the running component-wise sum of revealed El Gamal
  ciphertexts. See [Vote Chain].

Final VCT root

: The root of a round's Vote Commitment Tree at the moment the round
  leaves the ACTIVE state. Every Vote Reveal Proof in the round is
  anchored to it. See [Round Lifecycle].

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

Key-share holder

: A holder of a share of a round's election authority private key,
  produced by the key ceremony. Not a validator; see [Ratification] and
  `draft-valargroup-shielded-voting-setup` [^voting-setup].

Option

: One of the labeled choices a proposal offers. A proposal has between
  2 and $N_{\mathsf{opt}}$ options. See [Proposals and Decisions].

Partial decryption

: A key-share holder's contribution to decrypting an aggregate
  ciphertext, accompanied by a proof that it was computed with the
  holder's share. See [Partial Decryption].

Poll runner

: The entity responsible for conducting a voting round; it chooses the
  snapshot and derives the round's roots. See [Snapshot Configuration].

Proposal

: A question put to voters in a voting round, with a fixed list of
  options. A round carries up to 15 proposals, identified by 1-indexed
  sequential integers. See [Proposals and Decisions].

Ratification

: A key-share holder's published statement, made by acknowledging its
  key share, that it holds a verified share for a round and will take
  part in its tally. See [Ratification].

Relay

: An untrusted store-and-forward service to which a voter MAY hand a
  finished Share Reveal Message for submission to the vote chain at a
  later time. A relay constructs no proofs and receives no witness
  material. See [Share Submission].

Reveal window

: The period, following the voting window, during which the vote chain
  accepts share reveal transactions for a round. It ends at the round's
  $\mathsf{reveal}\_\mathsf{end}\_\mathsf{time}$. See [Round Lifecycle].

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

: The set of times at which a client's $N_s$ share reveal messages are
  submitted, drawn as specified in [Submission Timing].

Validator

: An operator of the vote chain's consensus. Validators determine
  transaction inclusion (see [Transaction Inclusion]) and hold no
  election authority key shares.

VAN nullifier

: A nullifier derived from a VAN commitment and published when the VAN
  is consumed (to cast a vote or delegate), preventing double-spending
  of voting authority.

Verification key

: The public value $\mathsf{VK}_i = [\mathsf{sk}_i]\, G$ corresponding
  to key-share holder $i$'s share, derivable by anyone from the
  ceremony's published commitments. See
  [Election Authority Key Ceremony].

Vote Authority Note (VAN)

: A commitment inserted into the Vote Commitment Tree that represents
  spendable voting authority. A VAN binds a voting hotkey, a ballot
  count, a voting round identifier, and a proposal authority bitmask.

Vote chain

: The purpose-built chain on which delegation, vote, share reveal and
  ceremony transactions are recorded. See [Vote Chain].

Vote Commitment (VC)

: A commitment inserted into the Vote Commitment Tree that binds a
  voter's encrypted share distribution, proposal choice, and vote
  decision for a single proposal. A VC is created when a VAN is consumed
  to cast a vote.

Vote Commitment Tree (VCT)

: An append-only Poseidon Merkle tree maintained by the vote chain that
  stores both VANs and VCs as leaves. A separate VCT is maintained per
  voting round; trees from different rounds are fully isolated. VANs
  span every proposal in a round, so the tree is per round rather than
  per proposal.

Vote manager

: The on-chain role authorised to create voting rounds. See
  [Poll Creation].

Vote share

: One of $N_s$ encrypted portions of a voter's ballot count within a
  Vote Commitment. Each share is revealed independently for
  homomorphic accumulation.

Voting round

: A bounded period during which a set of proposals are open for voting.
  Each round is identified by a unique $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ and
  is associated with a pool snapshot, an election authority public key,
  and a set of proposals.

Voting window

: The period during which the vote chain accepts delegation and vote
  transactions for a round. It ends at the round's
  $\mathsf{vote}\_\mathsf{end}\_\mathsf{time}$. See [Round Lifecycle].


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

After the reveal window closes, anyone can publicly aggregate the
revealed El Gamal ciphertexts per proposal option. Holders of shares of
the election authority key — a set disjoint from the validators, among
whom the key was generated without ever existing whole — cooperate to
produce partial decryptions; the results are
combined via Lagrange interpolation and the aggregate total is publicly
verified. The tally itself reveals only aggregates. What a party holding
a threshold of key shares can recover beyond that, and what the parties
controlling block production can withhold, are stated in
[Privacy Implications] and [Transaction Inclusion] rather than assumed
away.

This ZIP also specifies the voting round as a consensus object: how a
round is anchored to a Zcash block, how its snapshot roots are derived,
how administrators attest to it, the key ceremony that produces its
election authority key, and the ratification by key-share holders that
gates the opening of voting.


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
  encrypted shares, each revealed independently over its own network
  connection at its own randomly drawn time, so that no party can group
  a voter's shares by content, timing or origin.
- **Homomorphic tallying.** Encrypted shares are accumulated on-chain
  via component-wise point addition. Only the aggregate total per
  proposal option is ever decrypted, and no decision appears in
  cleartext.

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
This holds against a chain observer, against a relay, and against a
coalition holding $t$ key shares: such a coalition can decrypt any
individual share, but decryption yields a value, not an association.
Whether two shares belong to one vote is not recoverable from any
content the protocol publishes. What remains is metadata — the time at
which each reveal is submitted and the network path it arrives by —
which [Share Submission] and [Submission Timing] address.

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
relay or validator logs. That is the purpose of the per-share network
isolation and memoryless scheduling required in [Share Submission] and
[Submission Timing], and of the rule that a relay never holds more than
one share of a vote.

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
that encrypt zero without $t$ key shares. Chain observers, validators
and relays therefore learn neither how any share voted nor the running
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
among the round's key-share holders (see
[Election Authority Key Ceremony]); each holder ends the ceremony with
a share and nobody, at any point, with the key. An adversary must obtain
at least $t$ shares to decrypt an individual share ciphertext, where $t$
and the holder set are fixed by the ceremony and published with the
round. Vote splitting does not substitute for that threshold: it bounds
what a coalition that does reach $t$ can learn about any one voter, by
ensuring that what it can decrypt cannot be grouped.

Relays are trusted for availability only. A relay receives a finished
Share Reveal Message — the same bytes the chain will record — together
with the time at which to submit it, and learns from that message
exactly what a chain observer learns, plus the network origin of the
client that handed it over and the requested submission time. A relay
holding one share of a vote can group nothing. A relay that receives
two shares of one vote could associate them by origin or by the pair of
requested times, which is why [Share Submission] requires one relay per
share and an independent network path per submission.

**Non-membership tree queries.** Obtaining exclusion proofs for the
nullifier non-membership tree during delegation requires a source for
that tree's leaves. A client that holds the nullifier set at the
snapshot height constructs its own exclusion proof and reveals nothing.
A client that queries a server for one reveals which nullifier it asked
about, and therefore which note it holds, unless that server's retrieval
protocol conceals the query.


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
- The aggregate tally is publicly verifiable: any party can confirm the
  homomorphic accumulation.
- The delegation phase is compatible with hardware wallets that support
  only the standard Orchard PCZT [^pczt] signing flow, without
  requiring firmware changes specific to the voting protocol.


# Non-requirements

- The vote chain's consensus engine, block structure, transaction
  encoding and API. The rules a round follows — creation, attestation,
  lifecycle, key ceremony, ratification and transaction inclusion — are
  in scope; see [Voting Round] and [Vote Chain].
- How operators are organised to run a deployment: roles, validator
  onboarding, nullifier service operation, deployment architecture, and
  audit procedures. These are specified in
  `draft-valargroup-shielded-voting-setup` [^voting-setup].
- The service interface of a relay, its availability and fault
  tolerance, and the relationship between relay operators and other
  roles are out of scope; they belong with the operational
  specification. The rules a client follows in handing messages to
  relays are in scope (see [Share Submission]), because they determine
  whether this ZIP's privacy claims hold.
- Post-quantum security of the El Gamal encryption layer is out of
  scope.
- Retrieval of nullifier non-membership proofs by clients that do not
  hold the nullifier set. The tree itself is specified in
  [^balance-proof]; how a client obtains a proof against it without
  revealing which nullifier it asked about is a deployment concern.


# High-level summary

This section is non-normative.

The protocol proceeds in four phases within a voting round.

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
frozen and its final root published. The voter's client constructs a
Vote Reveal Proof for each of its $N_s$ shares against that root. Each
proof opens one share as a vector of ciphertexts, one per option
position, without revealing which Vote Commitment it came from or which
position carries the share's value. The client submits each resulting
message to the vote chain over an independent network path at an
independently drawn time within the reveal window, either directly or
by handing the finished message to a relay that submits it later. The
chain accumulates the ciphertext vectors homomorphically.

**Phase 4: Tally.** After the reveal window closes, at least $t$
key-share holders produce partial decryptions of the aggregate
ciphertext per (proposal, option) pair. The partial decryptions are
stored on-chain and combined via Lagrange interpolation to recover the
total ballot count (via the bounded discrete-log recovery procedure
defined in [Decryption]). Correctness is publicly verifiable: anyone
can recompute the Lagrange combination from the on-chain partial
decryptions.


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

The keypair is generated afresh for each round by the key ceremony
specified in `draft-valargroup-shielded-voting-setup` [^voting-setup].
That document also specifies how $\mathsf{ea}\_\mathsf{sk}$ is shared
among key-share holders, and is the normative reference for the
threshold $t$ used in [Tally].

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

This is what allows the vote chain to aggregate revealed share
ciphertexts into a per-$(\mathsf{proposal}\_\mathsf{id}, j)$
accumulator for each option position $j$ without decrypting any of
them, and it is why no party needs $\mathsf{ea}\_\mathsf{sk}$ before
the reveal window closes.

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

Distribution of key shares to their holders uses ECIES [^ecies]
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
published verification key, as specified in
`draft-valargroup-shielded-voting-setup` [^voting-setup].


## Data Structures

### Voting Round Identifier

Each voting round is identified by a 32-byte
$\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$, derived
deterministically from the round's setup parameters by Poseidon
hashing. Validators and verifiers compute it from the same inputs
and check that it matches the on-chain identifier.

$$\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}} = \mathsf{Poseidon}\bigl(\mathsf{snapshot}\_\mathsf{height},\ \mathsf{bh}\_\mathsf{lo},\ \mathsf{bh}\_\mathsf{hi},\ \mathsf{ph}\_\mathsf{lo},\ \mathsf{ph}\_\mathsf{hi},\ \mathsf{vote}\_\mathsf{end}\_\mathsf{time},\ \mathsf{reveal}\_\mathsf{end}\_\mathsf{time},\ \mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root},\ \mathsf{nc}\_\mathsf{root}\bigr)$$

The hash uses the $\mathsf{ConstantLength}\langle 9 \rangle$ variant
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
  in little-endian byte order.
- $\mathsf{vote}\_\mathsf{end}\_\mathsf{time} \in \{ 0 .. 2^{64}-1 \}$
  — Unix timestamp (seconds) after which votes are no longer
  accepted, encoded as a Pallas base field element.
- $\mathsf{reveal}\_\mathsf{end}\_\mathsf{time} \in \{ 0 .. 2^{64}-1 \}$
  — Unix timestamp (seconds) after which share reveals are no longer
  accepted, encoded as a Pallas base field element.
- $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root} \in \{ 0 .. q_{\mathbb{P}}-1 \}$
  — root of the nullifier non-membership IMT at the snapshot
  height. MUST be a canonical Pallas base field element.
- $\mathsf{nc}\_\mathsf{root} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ —
  Ironwood pool note commitment tree root at the snapshot height.
  MUST be a canonical Pallas base field element.

$\mathsf{snapshot}\_\mathsf{blockhash}$ and
$\mathsf{proposals}\_\mathsf{hash}$ are split into two 128-bit
limbs because they are not necessarily canonical Pallas field
elements. $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ and
$\mathsf{nc}\_\mathsf{root}$ are themselves Poseidon-derived in
their respective trees and are therefore already canonical.

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
- $\mathsf{proposal}\_\mathsf{authority} \in \{0 \ldots 2^{16}-1\}$ — bitmask
  encoding which proposals this VAN is authorized to vote on.
  $\mathsf{proposal}\_\mathsf{id}$ values 1–15 map to bits 1–15; bit 0 is
  reserved (see [Why Proposal Identifiers Start at 1]). Full authority
  is $2^{16} - 1 = 65535$.
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
- $\mathsf{proposal}\_\mathsf{id} \in \{1 \ldots 15\}$ — which proposal this
  vote targets.
- $\mathsf{vote}\_\mathsf{decision} \in \{ 0 .. q_{\mathbb{P}}-1 \}$ — the
  voter's choice (0-indexed into the proposal's declared options).

A VC MUST be created during voting (Phase 2) and opened during share reveal (Phase 3).

The VC hash MUST be posted on-chain as a public input of
the Vote Proof and inserted into the VCT.

Its preimage fields
($\mathsf{shares}\_\mathsf{hash}$ and $\mathsf{vote}\_\mathsf{decision}$)
MUST be private witnesses in that proof.

During share reveal, the Vote
Reveal Proof MUST prove membership in the VCT without exposing which VC
is being opened, and MUST NOT expose $\mathsf{vote}\_\mathsf{decision}$.

### Vote Share

A vote share is one of $N_s = 16$ encrypted portions of a voter's ballot
count within a VC.

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

The share
nullifier MUST be posted on-chain as a public input of the Vote Reveal
Proof and added to the share nullifier set to prevent double-counting.

### Vote Commitment Tree

The vote chain MUST maintain a separate VCT per voting round. Each
round's VCT MUST be an incremental Merkle tree [^protocol-merkletree]
of depth $\mathsf{MerkleDepth}^{\mathsf{vct}} = 24$ that stores both
VANs and VCs as leaves. Leaves, roots, and tree state from one round
MUST NOT carry over into another. The tree MUST use the same
append-only data structure as the Orchard note commitment tree, but
with Poseidon over the Pallas scalar field for internal node hashing
instead of Sinsemilla (see [Poseidon Instantiation]).

Domain separation between VANs and VCs is achieved structurally: the
first Poseidon input is $\mathsf{DOMAIN}\_\mathsf{VAN} = 0$ for VANs and
$\mathsf{DOMAIN}\_\mathsf{VC} = 1$ for VCs, making it impossible for a valid VAN
preimage to produce the same hash as a valid VC preimage.

Leaves MUST be inserted in transaction order: a delegation transaction
inserts one VAN; a vote transaction inserts both a new VAN and a VC.

### Nullifier Sets

The vote chain MUST maintain three disjoint nullifier sets:

1. **Governance nullifiers**: prevent double-delegation of mainchain
   Ironwood pool notes within a voting round.
2. **VAN nullifiers**: prevent double-spending of voting authority.
3. **Share nullifiers**: prevent double-counting of revealed shares.

Each set SHOULD BE append-only within a voting round. The vote chain MUST reject any transaction that publishes a nullifier already present in the corresponding set.


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

The vote chain SHOULD NOT insert $\mathsf{cmx}\_\mathsf{new}$ into any commitment tree
or use it beyond proof verification; it serves only to make the
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

A verifier that receives a Delegation Proof $\pi$ together with a spend
authorization signature $\sigma$ MUST perform the following checks:

1. Verify $\pi$ against the public inputs.
2. Verify that $\mathsf{sighash}\_\mathsf{del}$ is exactly 32 bytes.
3. Verify $\sigma$ as a valid $\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$
   signature on $\mathsf{sighash}\_\mathsf{del}$, under $\mathsf{rk}$.
4. Verify that $\mathsf{rt}^{\mathsf{cm}}$ and $\mathsf{rt}^{\mathsf{excl}}$ correspond
   to the published pool snapshot for $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$.
5. Verify that no $\mathsf{gov}\_{\mathsf{null}\_\mathsf{i}}$ appears in the governance
   nullifier set. If any does, reject as a double-delegation.
6. Verify that $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ matches an active round.
7. Insert $\mathsf{van}$ into the VCT.
8. Add all $\mathsf{gov}\_{\mathsf{null}\_\mathsf{i}}$ to the governance nullifier set.

### Delegation Sighash

The delegation sighash $\mathsf{sighash}\_\mathsf{del}$ is a
client-provided 32-byte value included in the delegation message. For
hardware wallet flows, the client computes it as the ZIP 244 [^zip-244]
shielded transaction sighash of a governance PCZT [^pczt]; the
construction of this PCZT is specified
in [^balance-proof]. For software wallets, the client signs the
sighash directly without PCZT construction.

The vote chain verifies the
$\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$ against this
client-provided sighash without recomputing it. The sighash does not
need to be independently reconstructible by the vote chain because the
Delegation Proof provides the governance data binding: the ZKP proves
that $\mathsf{rk}$ is a valid rerandomization of the holder's spend
authorization key, that the holder owns the claimed notes, and that the
VAN commitment is correctly constructed. An attacker who substitutes a
different sighash cannot produce a valid signature under
$\mathsf{rk}$ without knowledge of the holder's spending key.

### Delegation Message

A delegation transaction submitted to the vote chain MUST contain:

| Field | Type | Description |
|---|---|---|
| $\pi\_\mathsf{del}$ | Proof | The Delegation Proof |
| $\sigma\_\mathsf{del}$ | Signature | SpendAuthSig under $\mathsf{rk}$ |
| $\mathsf{sighash}\_\mathsf{del}$ | 32 bytes | Client-computed sighash (see [Delegation Sighash]) |
| $\mathsf{signed}\_{\mathsf{note}\_\mathsf{nullifier}}$ | Pallas scalar | Dummy note nullifier |
| $\mathsf{rk}$ | Pallas point | Randomized verification key |
| $\mathsf{rt}^{\mathsf{cm}}$ | Pallas scalar | Note commitment tree root |
| $\mathsf{rt}^{\mathsf{excl}}$ | Pallas scalar | Non-membership tree root |
| $\mathsf{van}$ | Pallas scalar | VAN commitment |
| $\mathsf{gov}\_{\mathsf{null}\_1} \ldots \mathsf{gov}\_{\mathsf{null}\_5}$ | Pallas scalar each | Governance nullifiers |
| $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ | Pallas scalar | Round identifier |
| $\mathsf{cmx}\_\mathsf{new}$ | Pallas scalar | Output note commitment |


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
- $\mathsf{anchor}\_\mathsf{height} ⦂ \mathbb{N}$ — VCT snapshot height.
- $\mathsf{proposal}\_\mathsf{id} ⦂ \{1 \ldots 15\}$ — which proposal.
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
  16 boolean wires $b_0, \ldots, b_{15}$ that recompose to the original
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

A verifier that receives a Vote Proof $\pi$ together with a vote
spend authorization signature $\sigma$ MUST perform the following
checks:

1. Verify $\pi$ against the public inputs.
2. Compute the vote sighash from the message fields
   (see [Vote Sighash]).
3. Verify $\sigma$ as a valid $\mathsf{SpendAuthSig}^{\mathsf{Orchard}}$
   signature on the vote sighash, under $\mathsf{r}\_\mathsf{vpk}$.
4. Verify that $\mathsf{van}\_\mathsf{nullifier}$ does not appear in the VAN
   nullifier set. If it does, reject as double-voting.
5. Verify that $\mathsf{anchor}\_\mathsf{height}$ refers to a VCT state
   within the current voting round and that $\mathsf{rt}^{\mathsf{vct}}$
   matches the VCT root at that height.
6. Verify that $\mathsf{proposal}\_\mathsf{id}$ is valid for the current round
   and within the voting window.
7. Verify that $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ matches an active round.
8. Verify that $\mathsf{ea}\_\mathsf{pk}$ matches the published election
   authority public key for this round.
9. Insert $\mathsf{van}_{\mathsf{new}}$ and $\mathsf{vc}$ into the VCT.
10. Add $\mathsf{van}\_\mathsf{nullifier}$ to the VAN nullifier set.

Note: $\mathsf{vote}\_\mathsf{decision}$ is a private witness in the
Vote Proof and is never published. Its range is enforced when the share
is revealed, by condition 6 of the [Vote Reveal Proof]; a decision at
or beyond the proposal's option count is accumulated into a position
the tally never decrypts.

### Vote Sighash

The vote sighash is computed by the vote chain from the vote message
fields. Unlike the delegation sighash, it is not client-provided;
the governance hotkey is software-controlled and signs the same
chain-computable digest.

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
$\alpha_v$) before submitting the vote message.

### Vote Message

A vote transaction submitted to the vote chain MUST contain:

| Field | Type | Description |
|---|---|---|
| $\pi_{\text{vote}}$ | Proof | The Vote Proof |
| $\sigma_{\text{vote}}$ | Signature | SpendAuthSig under $\mathsf{r}_{\mathsf{vpk}}$ |
| $\mathsf{van}\_\mathsf{nullifier}$ | Pallas scalar | Old VAN nullifier |
| $\mathsf{r}_{\mathsf{vpk}}$ | Pallas point | Randomized voting public key |
| $\mathsf{van}_{\mathsf{new}}$ | Pallas scalar | New VAN commitment |
| $\mathsf{vc}$ | Pallas scalar | Vote commitment |
| $\mathsf{rt}^{\mathsf{vct}}$ | Pallas scalar | VCT root |
| $\mathsf{anchor}\_\mathsf{height}$ | integer | VCT anchor height |
| $\mathsf{proposal}\_\mathsf{id}$ | $\{1 \ldots 15\}$ | Proposal identifier |
| $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ | Pallas scalar | Round identifier |
| $\mathsf{ea}_{\mathsf{pk}}$ | Pallas point | EA public key |


## Share Reveal Phase

Share reveal takes place during the reveal window, after the voting
window has closed and the round's VCT has been frozen (see
[Round Lifecycle]). The voter's client constructs one Vote Reveal Proof
per share and submits the resulting messages as specified in
[Share Submission]. No party other than the voter's client constructs a
Vote Reveal Proof; see [Why Relays Do Not Construct Proofs].

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
- $\mathsf{proposal}\_\mathsf{id} ⦂ \{1 \ldots 15\}$ — which proposal.
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

A verifier that receives a Vote Reveal Proof $\pi$ MUST perform the
following checks:

1. Verify $\pi$ against the public inputs.
2. Verify that $\mathsf{share}\_\mathsf{nullifier}$ does not appear in the
   share nullifier set. If it does, reject as double-counting.
3. Verify that $\mathsf{rt}^{\mathsf{vct}}$ equals the round's final
   VCT root (see [Round Lifecycle]).
4. Verify that $\mathsf{proposal}\_\mathsf{id}$ is valid for the round.
5. Verify that $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ matches a
   round in the REVEALING state.
6. Verify that $\mathsf{ea}\_\mathsf{pk}$ matches the round's election
   authority public key.
7. Add $\mathsf{share}\_\mathsf{nullifier}$ to the share nullifier set.
8. For each position $j \in \{0 \ldots N_{\mathsf{opt}} - 1\}$,
   accumulate $E_j$ into the aggregate ciphertext for
   $(\mathsf{proposal}\_\mathsf{id}, j)$:

$$\mathsf{agg}[\mathsf{proposal}\_\mathsf{id}][j] \mathrel{+}= E_j$$

where $+$ denotes component-wise Pallas point addition.

The verifier does not learn, and does not check, which position carries
the share's value. A share whose committed decision is at or beyond the
proposal's option count is accumulated into a position that [Tally]
never decrypts; it contributes to no option, and nothing in this
protocol allows a misplaced share to alter another option's total.

### Share Reveal Message

A share reveal transaction submitted to the vote chain MUST contain:

| Field | Type | Description |
|---|---|---|
| $\pi_{\text{reveal}}$ | Proof | The Vote Reveal Proof |
| $\mathsf{share}\_\mathsf{nullifier}$ | Pallas scalar | Share nullifier |
| $E_0, \ldots, E_{N_{\mathsf{opt}} - 1}$ | $N_{\mathsf{opt}} \times (E_1, E_2)$ | Option-vector ciphertexts (two Pallas points each) |
| $\mathsf{proposal}\_\mathsf{id}$ | $\{1 \ldots 15\}$ | Proposal identifier |
| $\mathsf{rt}^{\mathsf{vct}}$ | Pallas scalar | Final VCT root |
| $\mathsf{voting}\_{\mathsf{round}\_\mathsf{id}}$ | Pallas scalar | Round identifier |

A share reveal message carries no signature and names no submitter. The
proof binds every field, so the message is valid regardless of who
submits it, and a relay that submits it on the voter's behalf adds
nothing to it and cannot alter it.

### Share Submission

A share reveal message reaches the vote chain either directly from the
voter's client or through a relay. In both cases the client constructs
the Vote Reveal Proof; the difference is only who transmits the
finished message and when.

**Constructing the messages.** Once the round enters REVEALING, the
client MUST obtain the round's final VCT root and a Merkle path for its
VC against that root, construct the Vote Reveal Proof for each share
$i \in \{0 \ldots N_s - 1\}$ per [Vote Reveal Proof], and assemble the
$N_s$ share reveal messages. A client MUST NOT send any of the
auxiliary inputs of the Vote Reveal Proof — the vote commitment, its
VCT position or path, the shares hash, the share commitments, the blind
factors or the committed ciphertexts — to any other party.

**Independence of submissions.** Whether submitted directly or through
relays, the $N_s$ messages of one vote MUST be submitted as if by $N_s$
unrelated clients:

1. Each message MUST be submitted over a network connection that shares
   no identifying state with the connection used for any other message
   of the same vote — for example, a separate Tor circuit or mixnet
   channel per message. A client MUST NOT reuse a circuit, a source
   address it controls the visibility of, or a session across two
   messages of one vote.
2. Each message MUST be submitted at a time drawn as specified in
   [Submission Timing].
3. A client MUST NOT submit two messages of one vote to the same relay,
   and MUST NOT submit a message to a relay that has already received a
   message of the same vote, including on retry.

**Direct submission.** The client submits each message itself, at its
scheduled time, subject to the rules above. Direct submission requires
the client to be online at each scheduled time.

**Relayed submission.** A client that will not be online for the
duration of its schedule MAY hand each message to a relay. For each
message, the client sends the relay a payload consisting of the
complete share reveal message and
$\mathsf{submit}\_\mathsf{at}$, the Unix time (seconds) at which the
relay is to submit it, with the value 0 meaning as soon as possible.
The client MUST select a distinct relay for each message, independently
and uniformly at random from the relays it is configured with, and MUST
hand each payload over under rule 1 above. The client SHOULD hand the
payloads to relays at independently drawn times rather than in one
burst, and MUST NOT hand them over in share-index order.

A relay receives only what the chain will record, and learns from the
payload nothing that a chain observer would not; see
[Privacy Implications]. It does learn the client's network origin as
presented and the requested submission time, which is why each message
goes to a different relay over a different path.

**Retry.** A client SHOULD confirm that each of its messages has been
included on the vote chain. A client that observes that a message has
not appeared within a client-configured timeout MAY resubmit it, to a
different relay or directly, under the rules above. Because a share
nullifier is accepted once, a duplicate submission is rejected by the
chain and is harmless. A client MUST NOT resubmit a message to a relay
that has already received it or any other message of the same vote.
Confirming inclusion by querying the chain for the client's own share
nullifiers reveals which nullifiers are the client's; see
[Open issues].

**Relay interface.** The relay's service interface is out of scope for
this ZIP. Whatever its form, a relay MUST accept a payload without
authenticating the submitting client, MUST NOT require any identifier
that persists across submissions, and MUST submit the message it was
given unaltered.

### Submission Timing

Share submission follows the scheduling discipline that ZIP 318
[^zip-0318] specifies for pool-crossing transfers. The two problems are
the same: a client emits several transactions that together reveal a
quantity it wishes to keep private, and an observer who can group them
recovers that quantity. ZIP 318 addresses it with randomized
decomposition, randomized ordering, and memoryless inter-arrival
delays. [Vote Share] already supplies the first. This section supplies
the other two.

Let $T_{\mathsf{end}}$ be the round's
$\mathsf{reveal}\_\mathsf{end}\_\mathsf{time}$, let $T_0$ be the time
at which the client commits a schedule, which is no earlier than the
start of the reveal window, and let $W = T_{\mathsf{end}} - T_0 -
\Delta$, where $\Delta$ is a deployment-specified safety margin
covering inclusion (see [Deployment]).

A client constructing a submission schedule:

1. MUST shuffle the $N_s$ shares into a uniformly random order before
   assigning submission times, so that the sequence of share values a
   client emits is not a function of their magnitudes or of their
   indices within the vote commitment.
2. MUST assign submission times by advancing a running offset from
   $T_0$, drawing each successive delay independently from an
   exponential distribution with rate
   $\lambda = 1 / \mathsf{MEAN}\_\mathsf{DELAY}$, where
   $\mathsf{MEAN}\_\mathsf{DELAY} = W / (N_s + 1)$.
3. MUST discard and redraw any delay exceeding
   $\mathsf{MAX}\_\mathsf{DELAY}$ (see [Deployment]).
4. MUST NOT impose a minimum separation between consecutive draws.
   Enforcing one would destroy the memorylessness that makes the
   schedule uninformative; occasional short gaps are a property of the
   distribution, not a defect.
5. MUST draw all randomness used in the shuffle and the delays from a
   cryptographically secure random number generator.

On the relayed path, the drawn times are the
$\mathsf{submit}\_\mathsf{at}$ values handed to the relays. The
schedule protects the on-chain footprint and the relays' view alike,
because no relay holds more than one message of a vote.

**When the window is short.** If the accumulated schedule would place
any share after $T_{\mathsf{end}} - \Delta$, the client MUST compress
the schedule by drawing each remaining share's submission time
independently and uniformly from the interval
$[\mathsf{now}, T_{\mathsf{end}} - \Delta]$. A client MUST NOT submit
the remaining shares as a batch, simultaneously, or in share-index
order, and MUST NOT place more than one share into a single vote chain
block where it can observe block boundaries. Where the remaining window
is too short for the client to submit all $N_s$ shares at all, the
client MUST inform the voter that the round is closing and that
proceeding will submit shares in close succession, rather than
proceeding silently.

Submitting promptly is not a substitute for submitting independently. A
client that reacts to a closing window by sending everything at once
reproduces, through timing, the exposure that
[Why There Is No Single-Share Mode] removes from the payload.

There is no single-share submission mode. Earlier drafts specified that
a voter casting within a final window place their entire ballot count
into one share, submitted immediately. A client MUST NOT do this: it
concentrates the voter's entire weight into one ciphertext, so a single
decryption recovers it exactly. See
[Why There Is No Single-Share Mode].


## Tally

After the reveal window closes, each per-$(\mathsf{proposal}\_\mathsf{id},
j)$ aggregate ciphertext, for each option position $j$ below the
proposal's option count, is decrypted by a threshold procedure. No
party reconstructs $\mathsf{ea}\_\mathsf{sk}$ at any point.

Let $t$ be the round's decryption threshold, let $\mathsf{QUAL}$ be
the round's key-share holder set, and let each holder
$i \in \mathsf{QUAL}$ hold the share $\mathsf{sk}_i$ of
$\mathsf{ea}\_\mathsf{sk}$, with public verification key
$\mathsf{VK}_i = [\mathsf{sk}_i]\, G$, as produced by
[Election Authority Key Ceremony]. The shares are Shamir shares
[^shamir] on a polynomial of degree $t - 1$, so any $t$ of them suffice.

### Aggregation

For each $(\mathsf{proposal}\_\mathsf{id}, j)$ pair, the aggregate
ciphertext $(C_{1,\mathsf{agg}}, C_{2,\mathsf{agg}})$ is the
component-wise sum of the position-$j$ ciphertext of every share reveal
accepted for that proposal, per [Additive Homomorphism]. Aggregation is
publicly verifiable: anyone holding the chain's share reveal
transactions can replay it.

Positions at or beyond the proposal's option count accumulate the
padding ciphertexts of every reveal and the committed ciphertext of any
share whose decision was out of range. They MUST NOT be decrypted and
are not part of the tally.

### Partial Decryption

At least $t$ key-share holders each publish a partial decryption

$$D_i = [\mathsf{sk}_i]\, C_{1,\mathsf{agg}}$$

Each $D_i$ MUST be accompanied by a Chaum-Pedersen DLEQ proof, as
specified in [Chaum-Pedersen DLEQ Proofs], instantiated with
$P = \mathsf{VK}_i$, $H = C_{1,\mathsf{agg}}$, $Q = D_i$ and witness
$x = \mathsf{sk}_i$. The proof demonstrates
$\log_G(\mathsf{VK}_i) = \log_{C_{1,\mathsf{agg}}}(D_i)$, establishing
that the share behind the holder's published verification key is the
share used to compute $D_i$.

A partial decryption whose proof does not verify MUST be rejected and
MUST NOT be included in the combination below. Without this check a
single holder could publish a bogus $D_i$, and the resulting
combination would yield a point whose discrete logarithm search fails
or returns an unrelated value, with no indication of which holder was
responsible.

### Combination

Given verified partial decryptions $\{(i, D_i)\}$ from a set $S$ with
$|S| \ge t$, the Lagrange coefficients at $0$ are

$$\lambda_i = \prod_{j \in S,\, j \neq i} \frac{-j}{i - j}$$

and the combination in the exponent recovers

$$[\mathsf{ea}\_\mathsf{sk}]\, C_{1,\mathsf{agg}} = \sum_{i \in S} [\lambda_i]\, D_i$$

from which the aggregate plaintext follows by [Decryption]:

$$[\mathsf{total}\_\mathsf{value}]\, G = C_{2,\mathsf{agg}} - [\mathsf{ea}\_\mathsf{sk}]\, C_{1,\mathsf{agg}}$$

$\mathsf{total}\_\mathsf{value}$ is then recovered by baby-step giant-step
and published for that $(\mathsf{proposal}\_\mathsf{id}, j)$ pair as
the total for option $j$, in the units specified in [Tally units].

Only the aggregate is decrypted. No step of this procedure reveals an
individual vote amount — but see [Privacy Implications] for what a
coalition holding $t$ shares can do outside this procedure, and
[Verification] for what a published tally does and does not establish.


## Voting Round

A voting round is the unit within which delegation, voting, share
reveal and tally take place. This section specifies what a round is —
its proposals and its Zcash snapshot — and the consensus rules by which
the vote chain creates, opens, and closes one: creation, attestation,
the lifecycle states, the election authority key ceremony that produces
the round's key, and the ratification that gates the transition to
voting. Who performs each step, and how operators are organised to do
so, is specified in `draft-valargroup-shielded-voting-setup` [^voting-setup].

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
sequential indices.

#### Decisions

A **decision** is a voter's chosen option for a specific proposal,
represented as the option's 0-indexed position within that proposal's
option list. A decision is never published; each share reveal carries
one ciphertext per option position, and the encrypted share accumulator
is keyed by `(proposal_id, option position)`. See [Vote Chain] for the
accumulator and [Vote Reveal Proof] for the construction.

#### Kinds of polls that can be expressed

A voting round can carry **1 to 15 independent proposals**, and each
proposal can offer **2 to 8 labeled options**. This is sufficient for:

- Yes/no questions ("Approve proposal X?" with options Yes / No).
- Multiple-choice preference questions (for example, choosing among
  named candidates or funding tiers).
- Rating-style questions using a fixed option ladder.

The following ballot shapes are out of scope for this specification:

- Free-form write-in answers.
- Ranked-choice or weighted-ranking ballots.
- More than 15 proposals in a single voting round, or more than 8
  options in a single proposal.

The 15-proposal upper bound is imposed by the zero-knowledge vote
authority bitmask (1 bit is reserved as a sentinel). Polls requiring
more proposals or richer ballot structures are split across multiple
rounds or expressed through an external layer.


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
created, the round MUST NOT open, and a round already open MUST be
abandoned: the snapshot it is anchored to no longer exists, and the
eligibility of every vote cast in it is undefined.

Choosing the snapshot is the start of round setup, not a single
automatic action. The poll runner is responsible for the following
coordinated activities:

1. **Determine the snapshot roots.** The Ironwood pool note commitment
   tree root ($\mathsf{nc}\_\mathsf{root}$) and the nullifier non-membership
   tree root ($\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$) are
   the two roots $\mathsf{rt^{cm}}$ and $\mathsf{rt^{excl}}$ of the pool
   snapshot at height $H$, as defined in the "Pool Snapshot" section of
   `draft-valargroup-orchard-balance-proof` [^balance-proof], on
   the chain whose block at height $H$ has hash
   $\mathsf{snapshot}\_\mathsf{blockhash}$. No party has discretion over
   their values. The poll runner derives them by the procedure in
   [Snapshot Derivation].

2. **Ensure the nullifier service has the snapshot's PIR
   database.** The poll runner coordinates with each nullifier
   service operator (see the "Nullifier Service Operator" section of
`draft-valargroup-shielded-voting-setup` [^voting-setup]) so that
   their ingest and export pipelines (see the "Nullifier Service" section of
`draft-valargroup-shielded-voting-setup` [^voting-setup])
   have run to the chosen height before the round opens, so
   wallets can query exclusion proofs against that snapshot.

3. **Use the values during chain bootstrap.**
   $\mathsf{nc}\_\mathsf{root}$ and
   $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ are passed
   into the genesis state (see the "Genesis Validator Setup" section of
`draft-valargroup-shielded-voting-setup` [^voting-setup]) and
   into the voting round initialization transaction (see
   [Poll Creation]) so that on-chain verifiers and wallet
   clients use them as ZKP public inputs.

### Snapshot Derivation

This procedure defines what it means for a round's snapshot roots to be
correct, and how a party obtains them. Both roots are properties of
Zcash mainnet state at $(H, \mathsf{snapshot}\_\mathsf{blockhash})$, and
both are read from a Zcash consensus node that has validated the chain
to at least that height. No party reconstructs either tree from the
Zcash chain or from its leaves.

1. Confirm that the block at height $H$ on the node's best chain has
   hash $\mathsf{snapshot}\_\mathsf{blockhash}$. If it does not, the
   round is not well formed.
2. Obtain $\mathsf{nc}\_\mathsf{root}$: the Ironwood pool note
   commitment tree root as of the end of block $H$. This is Zcash
   consensus data. A node computes it while validating the chain and
   exposes it as the pool's anchor at that height.
3. Obtain $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$: the root
   of the nullifier non-membership tree over every Ironwood pool
   nullifier revealed at or before $H$, constructed as specified in
   `draft-valargroup-orchard-balance-proof` [^balance-proof].
   Zcash consensus does not commit to this tree; a node maintains it as
   an index over the nullifier set it already tracks, and serves its
   root.
4. Compare both values with the round's $\mathsf{nc}\_\mathsf{root}$
   and $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$.

A round is **well formed** if all four steps succeed.

**Who performs it.** The poll runner MUST perform it before publishing a
round's roots. Each administrator MUST perform it against a node under
its own control before attesting to a round (see [Round Attestation]).
Any party MAY perform it. Wallets are not required to: a wallet
authenticates the round configuration and binds it to the chain round as
specified in `draft-valargroup-shielded-voting-wallet-api`
[^wallet-api], and derives no roots of its own.

**Node requirements.** A deployment MUST identify the Zcash node
implementations and versions it relies on to serve
$\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ (see [Deployment]).
Because that root is not consensus data, two node implementations can
disagree about it with no Zcash consensus rule to settle the
disagreement. The construction specified in
`draft-valargroup-orchard-balance-proof` [^balance-proof] is the
sole authority: a party that obtains differing roots for the same
$(H, \mathsf{snapshot}\_\mathsf{blockhash})$ from independent nodes MUST
treat the round as not well formed until the discrepancy is resolved.

The served root is correct only if the tree behind it covers every
Ironwood pool nullifier revealed at or before $H$ and nothing else. A node
maintaining that index incrementally MUST:

- track the block hash at which each nullifier entered the index, not
  only its height;
- on a chain reorganisation, roll the index back to the last block
  common to the old and new best chains before applying the new blocks;
  and
- refuse to serve a root for $H$ until it has confirmed that the block
  it ingested at $H$ is on its best chain and has hash
  $\mathsf{snapshot}\_\mathsf{blockhash}$.

An index built by height alone can omit nullifiers from blocks that
replaced reorganised ones, or retain nullifiers from blocks no longer on
the best chain. Either produces a root that is wrong without any party
intending it, and an incomplete set is the condition under which a spent
note can be proven unspent.

### Poll Creation

The vote manager initializes the chain's voting round by submitting
a transaction carrying the client-supplied subset of the `VoteRound`
structure specified in `draft-valargroup-shielded-voting-wallet-api`
[^wallet-api]. The vote manager supplies `snapshot_height`,
`snapshot_blockhash`, `proposals_hash`, `vote_end_time`,
`reveal_end_time`, `nullifier_imt_root`, `nc_root`, `proposals`,
`title`, and
`description`; the transaction's signer becomes the `creator` field
of the resulting `VoteRound`. The chain derives the remaining
fields (`vote_round_id`, `status`, `ea_pk`, `created_at_height`)
at inclusion or during the round lifecycle.

The chain rejects the transaction if the signer is not the current
vote manager, if the `proposals` field violates the constraints
in [Proposals and Decisions], or if `reveal_end_time` is not later
than `vote_end_time` by at least $2\Delta$ (see [Round Lifecycle]).

The vote chain derives the 32-byte `vote_round_id` from the
transaction fields after inclusion, using the Poseidon construction
specified in [Voting Round Identifier]. The result is a Pallas field element
so that the round ID can enter ZKP circuits as a public input.

The round enters the **PENDING** state. The EA key ceremony (see
[Election Authority Key Ceremony]) runs automatically. Once it
completes and at least $t$ key-share holders have ratified the round
(see [Ratification]), the round transitions to **ACTIVE**, the voting
window opens, and the transition timestamp is recorded as
`ceremony_phase_start`. The round transitions to **REVEALING** at
`vote_end_time` and to **TALLYING** at `reveal_end_time` (see
[Round Lifecycle]). Clients construct their share submission schedule
within the reveal window, as specified in [Submission Timing]. There
is no last-moment buffer: the single-share mode that earlier drafts
defined for the end of the voting window has been removed, and a client
near the deadline compresses its schedule rather than concentrating its
weight.

### Round Attestation

Administrators attest to a round by signing its defining fields. A
wallet accepts a round only with at least $m$ valid attestations from
administrators it recognises, as specified in
`draft-valargroup-shielded-voting-wallet-api` [^wallet-api], and
the administrator signature threshold $m$ MUST be at least 2.

The bytes covered by an attestation are the concatenation, in this
order, of:

| Component | Width |
|---|---|
| The ASCII string `ZcashVotingRoundAttestation:v3` | 30 bytes |
| `vote_round_id` | 32 bytes |
| `snapshot_height`, big-endian unsigned | 4 bytes |
| `snapshot_blockhash` | 32 bytes |
| `nc_root` | 32 bytes |
| `nullifier_imt_root` | 32 bytes |
| `proposals_hash` | 32 bytes |
| `vote_end_time`, big-endian unsigned | 8 bytes |
| `reveal_end_time`, big-endian unsigned | 8 bytes |
| `ea_pk` | 32 bytes |
| `min_confirmations`, big-endian unsigned | 4 bytes |

All components are fixed width, so the encoding is unambiguous without
length prefixes. The domain separator distinguishes these bytes from any
other signature the same key may produce, and its version is that of
the vote configuration format that carries the attestation.

An administrator MUST NOT sign a round unless it has performed
[Snapshot Derivation] for that round, using a Zcash full node under
its own control, and obtained the round's $\mathsf{nc}\_\mathsf{root}$
and $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$. Agreement with
another party's copy of the configuration does not satisfy this:
comparing two copies of the same values establishes only that two
parties received the same input, not that the input is correct. See
[Why Snapshot Roots Are Independently Derived].

### Round Lifecycle

1. **PENDING**: round created, awaiting the EA key ceremony and its
   ratification by key-share holders (see [Ratification]).
2. **ACTIVE**: ceremony complete and round ratified; the voting window
   is open. The chain accepts delegation and vote transactions for the
   round and rejects share reveal transactions.
3. **REVEALING**: `vote_end_time` has passed; the reveal window is
   open. On entering this state the chain MUST record the VCT root as
   the round's **final VCT root**, and MUST thereafter reject delegation
   and vote transactions for the round, so that the VCT does not change.
   The chain accepts share reveal transactions anchored to the final
   root (see [Vote Reveal Proof]). Voters construct and submit their
   share reveal messages (see [Share Submission] and
   [Submission Timing]).
4. **TALLYING**: `reveal_end_time` has passed. Key-share holders submit
   partial decryptions, the chain combines them, and tally
   decryption runs automatically (see [Tally]). The
   chain enforces a bounded timeout on the TALLYING state: if a
   tally is not submitted within this timeout, the round
   auto-finalizes with no tally, preserving liveness.
5. **FINALIZED**: tally published and verifiable. A round that
   auto-finalized due to a TALLYING timeout publishes no tally.

`reveal_end_time` MUST be later than `vote_end_time` by at least
$2\Delta$ (see [Deployment]). A deployment MUST publish the reveal
window length it uses, and SHOULD choose one long enough that a voter
who opens their wallet at ordinary intervals does so at least once
within it; see [Why a Separate Reveal Window].

### Election Authority Key Ceremony

Each round uses a fresh election authority keypair
$(\mathsf{ea}\_\mathsf{sk}, \mathsf{ea}\_\mathsf{pk})$. The ceremony
that produces it is a distributed key generation (DKG) among the
round's key-share holders, run over the vote chain, which serves as
the ceremony's authenticated broadcast channel. It runs automatically
when the round enters PENDING. The construction is Pedersen's DKG
[^pedersen-dkg] with Feldman verifiable secret sharing [^feldman] and
a proof of knowledge of each contribution, in the form analysed in
[^gjkr] and adopted by FROST [^frost].

No party holds $\mathsf{ea}\_\mathsf{sk}$ at any point. Each holder
contributes a random polynomial; the key is the sum of the constant
terms, and each holder's share is the sum of the other holders'
evaluations at its index. Scoping the key to one round bounds the
damage from a share compromise to that round, and means a holder that
leaves the set cannot decrypt later rounds. The cryptographic
constructions used below are specified in
[El Gamal Encryption on Pallas], [ECIES on Pallas] and
[Chaum-Pedersen DLEQ Proofs].

**Participants.** The round's key-share holders are the parties
registered as such for the round, each with a registered Pallas public
key for receiving encrypted shares. Let $n$ be their number and index
them $1, \ldots, n$. The round's decryption threshold is
$t = \lceil n/2 \rceil + 1$, with a minimum of 2. How holders are
admitted and how they register keys is specified in
`draft-valargroup-shielded-voting-setup` [^voting-setup]; see
[Open issues] for what that document does not yet specify. A key-share
holder MUST NOT be a validator of the same round.

**Round 1: commitment.** Each holder $i$:

1. Samples $t$ coefficients $a_{i,0}, \ldots, a_{i,t-1}$ uniformly at
   random from the Pallas scalar field, defining
   $f_i(x) = \sum_{k=0}^{t-1} a_{i,k}\, x^k$.
2. Computes the Feldman commitments $A_{i,k} = [a_{i,k}]\, G$ for
   $k = 0, \ldots, t-1$.
3. Computes a Schnorr proof of knowledge of $a_{i,0}$: samples $k_i$
   at random, sets $R_i = [k_i]\, G$,
   $c_i = \mathsf{BLAKE2b\text{-}256}(\texttt{"svote-dkg-pok-v1"} \mathbin\| \mathsf{vote}\_\mathsf{round}\_\mathsf{id} \mathbin\| i \mathbin\| A_{i,0} \mathbin\| R_i)$
   reduced to a Pallas scalar, and $\mu_i = k_i + c_i \cdot a_{i,0}$.
4. Publishes to the chain, in a ceremony transaction: $A_{i,0}, \ldots,
   A_{i,t-1}$ and $(R_i, \mu_i)$.

The chain MUST reject a Round 1 transaction whose proof of knowledge
does not verify, that is, unless
$[\mu_i]\, G = R_i + [c_i]\, A_{i,0}$. The proof of knowledge prevents
a holder from choosing its commitment as a function of others' and so
biasing or cancelling the key; see [Why Distributed Key Generation].

Round 1 closes when every holder has published or the commitment
timeout has elapsed. Let $\mathcal{C}$ be the set of holders that
published a valid Round 1 transaction. If $|\mathcal{C}| < t$, the
ceremony fails and restarts.

**Round 2: dealing.** Each holder $i \in \mathcal{C}$ computes
$f_i(j)$ for every $j \in \mathcal{C}$, $j \neq i$, encrypts each to
holder $j$'s registered Pallas key using ECIES with a fresh ephemeral
scalar per recipient, and publishes the encrypted shares to the chain
in a ceremony transaction. A holder MUST then erase
$a_{i,1}, \ldots, a_{i,t-1}$ and every $f_i(j)$ for $j \neq i$,
retaining only $f_i(i)$. Round 2 closes when every holder in
$\mathcal{C}$ has published or the dealing timeout has elapsed.

**Round 3: verification and complaints.** Each holder $j$ decrypts
each share $f_i(j)$ addressed to it and checks it against holder
$i$'s commitments:

$$[f_i(j)]\, G = \sum_{k=0}^{t-1} [j^k]\, A_{i,k}$$

If the check fails for some $i$, or holder $i$ published no share for
$j$, holder $j$ publishes a complaint against $i$ within the complaint
window. A holder $i$ that receives a complaint from $j$ MUST respond,
within the same window, by publishing $f_i(j)$ in the clear. The chain
MUST verify the published value against $A_{i,\cdot}$ by the equation
above. Holder $i$ is **disqualified** if it fails to publish a Round 2
transaction, fails to respond to a complaint, or responds with a value
that fails verification. A holder whose response verifies is not
disqualified, and the complaining holder uses the published value as
its share from $i$.

Let $\mathsf{QUAL} \subseteq \mathcal{C}$ be the holders not
disqualified. If $|\mathsf{QUAL}| < t$, the ceremony fails and
restarts. $\mathsf{QUAL}$ is the round's key-share holder set; a
holder outside it holds no share of the round's key and MUST NOT take
part in the tally.

**Key derivation.** On the close of Round 3 the chain computes and
records:

$$\mathsf{ea}\_\mathsf{pk} = \sum_{i \in \mathsf{QUAL}} A_{i,0}$$

$$\mathsf{VK}_j = \sum_{i \in \mathsf{QUAL}} \sum_{k=0}^{t-1} [j^k]\, A_{i,k} \quad \text{for each } j \in \mathsf{QUAL}$$

Each holder $j \in \mathsf{QUAL}$ computes its share

$$\mathsf{sk}_j = \sum_{i \in \mathsf{QUAL}} f_i(j)$$

and MUST verify that $[\mathsf{sk}_j]\, G = \mathsf{VK}_j$ before
acknowledging. The shares $\mathsf{sk}_j$ are Shamir shares [^shamir]
of $\mathsf{ea}\_\mathsf{sk} = \sum_{i \in \mathsf{QUAL}} a_{i,0}$ on
the degree-$(t-1)$ polynomial $\sum_{i \in \mathsf{QUAL}} f_i$, so
[Tally] applies to them unchanged. Every $\mathsf{VK}_j$ is computed
from published values and can be recomputed by any party.

**Acknowledgement.** Each holder $j \in \mathsf{QUAL}$ that has
verified its share submits an acknowledgement transaction carrying

$$\mathsf{SHA256}\bigl(\texttt{"ack"} \mathbin\| \mathsf{vote}\_\mathsf{round}\_\mathsf{id} \mathbin\| \mathsf{ea}\_\mathsf{pk} \mathbin\| \mathsf{holder}\_\mathsf{address}\bigr)$$

A holder MUST NOT acknowledge a share that fails the verification key
check. Committing to $\mathsf{ea}\_\mathsf{pk}$ keeps an
acknowledgement from carrying over to a round rekeyed after the fact;
committing to `vote_round_id` keeps it from carrying over to another
round under the same key. This acknowledgement is also the holder's
ratification of the round; see [Ratification].

**Confirmation.** The ceremony confirms when every holder in
$\mathsf{QUAL}$ has acknowledged, or, after the acknowledgement
timeout, when at least $t$ have. Holders in $\mathsf{QUAL}$ that did
not acknowledge retain a valid share and MAY still take part in the
tally, but have not ratified the round. If fewer than $t$ acknowledge
within the timeout, the ceremony fails and restarts.

Requiring $t$ acknowledgements before confirmation is deliberate: the
number of acknowledgements required to confirm is the same $t$ used for
threshold decryption. Were confirmation to require fewer, a round could
open that could never be tallied.

**Failure and restart.** A ceremony that fails restarts from Round 1
with fresh randomness. Holders that caused a failure by not
participating are excluded from the restarted ceremony. A deployment
SHOULD publish each holder's participation record; how persistent
non-participation is treated is an operational matter specified in
`draft-valargroup-shielded-voting-setup` [^voting-setup].

**Holder set changes.** A party joining the holder set during a round
receives no share for that round and waits for the next. A holder
leaving retains its share and cannot be compelled to delete it;
per-round keys bound what that share is worth, since it opens nothing
in any other round.

**Timing parameters.** A deployment MUST publish the commitment,
dealing, complaint and acknowledgement timeouts it applies.

### Ratification

Attestation ([Round Attestation]) establishes that a round's parameters
are correct. It does not establish that the round will be tallied. Those
are different parties: administrators configure a round, and key-share
holders decrypt its result. A round can be correctly configured, voted
in, and never opened.

A key-share holder ratifies a round with the acknowledgement it submits
during the key ceremony, as specified in
[Election Authority Key Ceremony]. Submitting it is the holder's
statement that it holds a verified share for the round and will take
part in its tally.

A round MUST NOT enter ACTIVE until at least $t$ distinct key-share
holders have ratified it, where $t$ is the round's decryption threshold.
Below $t$ the question does not arise: a round ratified by fewer than $t$
holders cannot be tallied even if every ratifying holder honours its
statement, so requiring $t$ makes the ratifications a statement that the
round is tallyable, not merely that some holders are willing. Because
ratifications are vote chain transactions, they are published with the
round.

A ratification is a statement of intent, not an enforceable commitment.
A holder can ratify and then decline to take part, and nothing in this
document prevents that. What ratification provides is that the decision
is made and published before voters commit their balances, rather than
discovered afterwards, and that a holder declining to tally a round it
ratified is visibly departing from a published statement. See
[Why Ratification Precedes Voting].

### Election Authority Key Custody

**Share generation.** The ceremony in [Election Authority Key Ceremony]
generates the key in distributed form: no party ever holds
$\mathsf{ea}\_\mathsf{sk}$, and every holder can verify its own share
against published commitments without trusting any other participant.
The claim that no single party holds the key therefore rests on the
ceremony's construction rather than on any party's promise to erase
anything. What each holder MUST erase is its own polynomial's
non-constant coefficients and the shares it dealt to others, as
specified in Round 2; retaining them does not expose the key, but does
expose other holders' shares from that dealer.

**Retention.** Each key-share holder MUST destroy its share once the
round is finalized and its tally published, and a deployment MUST
publish the retention period it applies. The encrypted shares of every
individual vote remain on the vote chain permanently, and their
encryption is not post-quantum, so retained shares are a live capability
against a permanent record of individual voters' share values, not a
dormant convenience. Retention is not needed for audit: the partial
decryptions and their DLEQ proofs are published on chain and can be
re-verified at any time without the key. A deployment that retains
shares nonetheless MUST state for how long, and MUST treat that period
as the period over which its amount-privacy claims hold.


## Vote Chain

The vote chain MUST maintain the following state per voting round:

- The **Vote Commitment Tree** as defined in [Vote Commitment Tree].
- Three disjoint **nullifier sets** as defined in [Nullifier Sets].
- A **per-$(\mathsf{proposal}\_\mathsf{id}, j)$ encrypted share
  accumulator** for each option position
  $j \in \{0 \ldots N_{\mathsf{opt}} - 1\}$: the running component-wise
  sum of the position-$j$ ciphertexts of revealed shares.
- The round's **final VCT root**, recorded on entering REVEALING (see
  [Round Lifecycle]).

For each transaction type, the vote chain MUST verify the
corresponding proof and perform the out-of-circuit checks specified in
[Delegation Proof], [Vote Proof], or [Vote Reveal Proof] respectively.

The vote chain's block structure, transaction encoding and API are out
of scope for this ZIP. The rules by which it admits transactions — and
the capabilities that gives the parties who control block production —
are not, and are specified below.

### Transaction Inclusion

The vote chain is a CometBFT chain, and its validators determine which
transactions enter blocks. This section states the consequences for a
voting round, which are not otherwise recorded in this or any companion
specification.

**Admission by state.** The chain MUST accept delegation and vote
transactions for a round only while the round is ACTIVE, share reveal
transactions only while it is REVEALING, and partial decryptions only
while it is TALLYING; see [Round Lifecycle].

**What validators can do.** Validators controlling enough stake to
control block production can decline to include share reveal
transactions. A share reveal exposes its proposal identifier but not
its decision (see [Vote Reveal Proof]), and per-option totals are not
public while a round is open, so validators acting alone cannot select
reveals to exclude by the option they support. They can exclude reveals
by proposal, by time of arrival, by network origin, or wholesale. A
coalition of validators and $t$ key-share holders could decrypt reveals
as they arrive and exclude by option; the requirement that the two sets
be disjoint (see [Election Authority Key Ceremony]) exists to keep that
coalition from being a single organisation.

**What validators cannot do.** They cannot create votes, alter the
weight of an included vote, or misreport the tally: each is prevented
by a proof that any observer can check. Exclusion only removes support;
it cannot manufacture it.

**What this means for a result.** A published result is a lower bound
on the support each option received, not a measurement of it. Results
SHOULD be described in those terms.

Separately from what validators can do, whether a result may be
described as representative of coinholder sentiment also depends on
conditions on the round itself, such as the diversity of wallet
implementations available to voters. Those conditions are not yet
specified; see [Open issues].

**Detection.** Exclusion is detectable but not provable from chain
state alone. An excluded transaction leaves no record on the chain that
excluded it. Available signals are: a count of votes cast against the
count of shares revealed; the contents of honest nodes' mempools,
compared with what was subsequently included; and voters observing
that their own shares never appeared. The last is currently unavailable
in practice, because a voter querying the chain for their own share
nullifiers reveals which nullifiers are theirs; see [Open issues].

A deployment SHOULD publish, for each round, the count of share reveal
transactions accepted into the mempool alongside the count included in
blocks, from more than one operator, so that a discrepancy is visible
without requiring any party to be trusted.

**Threshold.** A deployment MUST publish the stake distribution across
validators for a round, and the proportion of stake required to control
block production, so that the size of the coalition required to exclude
transactions is a published figure rather than an inferred one.



### Poseidon Instantiation

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
| Voting round identifier | 9 | $\mathsf{ConstantLength}\langle 9 \rangle$ | 5 |
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

### Domain Separator Tags

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

A verifier with a copy of the vote chain MUST check each of the
following. They are ordered so that each step presupposes the ones
above it.

1. **Round configuration.** The round's snapshot roots are correct, as
   established by the procedure in [Snapshot Derivation]. This is
   not verifiable from vote chain state, because the roots are supplied
   as input at round creation rather than derived by consensus. A
   verifier that omits this step establishes only that votes are well
   formed *with respect to* roots it has not checked.
2. **Transaction validity.** Every delegation, vote, and share reveal
   transaction in the round carries a valid proof, and satisfies the
   out-of-circuit checks in [Delegation Proof], [Vote Proof] and
   [Vote Reveal Proof] respectively.
3. **Nullifier disjointness.** The three nullifier sets defined in
   [Nullifier Sets] contain no duplicates, so no voting authority was
   consumed twice.
4. **Accumulation.** Each per-$(\mathsf{proposal}\_\mathsf{id}, j)$
   accumulator equals the component-wise sum of the position-$j$
   ciphertexts in the round's share reveal transactions, and every
   share reveal is anchored to the round's final VCT root.
5. **Decryption.** The published per-option totals are the decryptions
   of those accumulators, as attested by the threshold decryption
   proofs specified in [Partial Decryption].
6. **Unit.** The published figures are interpreted in the unit in which
   they are denominated; see [Ballot Scaling] and [Tally units].

**What this establishes.** Steps 2 through 5 establish that the totals
are the correct sum of the votes present on the chain. With step 1,
they establish that those votes were cast by holders of the balances
they claim.

**What it does not establish.** No step above, and no combination of
them, establishes that every vote cast was included. Exclusion of a
share reveal transaction leaves no evidence in chain state; see
[Transaction Inclusion]. A verified result is therefore a lower bound on the
support each option received.

**On partial verification.** Checking step 5 alone confirms only that
the announced totals match the accumulators — it does not check any
proof, and it does not check either snapshot root. A tool or procedure
that performs only the decryption check MUST NOT be described as
verifying a round's result. Implementations of verification tooling
SHOULD state which of the steps above they perform.


# Rationale

## Why a Separate Vote Chain

Voting transactions (delegation proofs, vote proofs, share reveals) are
not standard Zcash shielded transactions. They require a new commitment
tree (the VCT), new nullifier sets, and an encrypted share accumulator
with homomorphic aggregation. Implementing these as a sidechain avoids
modifying the Zcash consensus layer and keeps the governance mechanism
independent of mainchain upgrade cycles.

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
removed; see [Why Relays Do Not Construct Proofs].

With content linkage removed, the remaining channel is metadata:
submission time and network origin. Those are addressed by the
memoryless schedule in [Submission Timing] and the per-share network
isolation and one-relay-per-share rules in [Share Submission].

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

## Why Relays Do Not Construct Proofs

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

What a server was genuinely providing was availability: a wallet that
is not online at each scheduled submission time needs some party to
submit for it. That is a store-and-forward function, and it does not
require the proof to be built by the party that forwards it. A relay
that receives a finished share reveal message holds exactly what the
chain will hold, and can group nothing that a chain observer could not.
The temporal mixing that server-side construction was meant to enable
is now provided by the schedule the client draws and the relay honours,
with the difference that the relay is no longer in a position to defeat
it.

Content linkage had to be removed before timing and network measures
had anything to protect: a server told which shares belong together
does not need to infer it. With the payload reduced to the message
itself, those measures are effective, and [Share Submission] requires
them.

## Why One Relay Per Share

Earlier drafts required each share to be sent to
$\lceil s/2 \rceil$ of the $s$ available servers, for censorship
resistance through redundancy, and later drafts capped the number of a
vote's shares any one server could receive. Both rules were reasoning
about a server that could read the association between shares from
the payload, and neither is the right rule once it cannot.

A relay that holds one share of a vote learns nothing about the vote's
other shares from any source. A relay that holds two learns nothing
from their contents, but does hold two arrival events with their
network origins and two requested submission times, and can group them
by any of those if the client was careless. The rule that follows is
the simplest one: exactly one share per relay, each handed over on its
own network path. It makes the relay's view of any vote a single
message, which is the same as any observer's view of any single
transaction.

Censorship resistance is recovered through client-side retry: a client
that does not observe its share on chain resubmits it, directly or to
a relay that has not seen any of its shares. Redundancy is obtained
sequentially, on demand, rather than prophylactically to half the
fleet, and a duplicate that does reach the chain is rejected by its
nullifier and does no harm.

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
relays, validators and chain observers, and against any party holding
fewer than $t$ key shares. A coalition of $t$ holders that decrypts an
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

Two differences from ZIP 318 are deliberate. First, delays here are
expressed in wall-clock time against the round's
$\mathsf{reveal}\_\mathsf{end}\_\mathsf{time}$ rather than in Zcash
block deltas, because the deadline is a vote chain parameter and the
vote chain's block rate is not the Zcash block rate. Second,
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

The $\mathsf{proposal}\_\mathsf{authority}$ bitmask is 16 bits wide, but
$\mathsf{proposal}\_\mathsf{id}$ values start at 1, yielding 15 usable
proposal slots rather than 16. Bit 0 is reserved.

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
that compromise of any set of holders smaller than $t$ does not expose
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
sum, and every downstream step — verification keys, partial
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


## Why Snapshot Roots Are Independently Derived

Requiring more administrators to sign a round does not, on its own, make
its snapshot roots any more likely to be correct. If no signer obtains
the roots independently, a threshold of signatures attests only that
several parties received the same document from the same source. An
incorrect nullifier root is not a remote failure: if the set behind it
omits a nullifier revealed on Zcash mainnet before the snapshot, the
holder of the spent note can prove it unspent and vote with it as well
as with the note that replaced it, and an index that mishandles a chain
reorganisation can produce such a set with no malice involved.
Independent derivation is what makes attestation meaningful: a signature
threshold multiplies an underlying check, and without the check there is
nothing to multiply.

Independence here is a property of the source, not of the effort. Each
administrator reads both roots from a Zcash node under its own control,
rather than accepting values supplied by the party proposing the round.
Because the node already serves as that administrator's oracle for Zcash
state — it is trusted for the block hash at $H$ and for
$\mathsf{nc}\_\mathsf{root}$ regardless — serving
$\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ alongside them adds
no trust assumption that attesting did not already carry. It does add a
dependency on node implementations maintaining that index, which
[Snapshot Derivation] requires a deployment to name.

## Why Bind to a Block Hash

Anchoring a round to a height alone leaves its meaning dependent on
which chain the reader follows. Binding it to
$(H, \mathsf{snapshot}\_\mathsf{blockhash})$ makes a reorganisation
affecting the snapshot a detectable condition with a specified response,
rather than a silent change in the eligible note set.

## Why Ratification Precedes Voting

A voter deciding whether to take part is deciding whether to expose a
quantity — their balance at the snapshot, to the extent the protocol
permits — in exchange for influence over an outcome. That trade exists
only if the outcome will be produced. A round names
$\mathsf{ea}\_\mathsf{pk}$, but a public key is not a statement by
anybody that they will use the corresponding shares. A statement
collected after the round records what happened; one collected before it
opens is an input the voter can act on.

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
come online once during the reveal window, construct the proofs, and
either submit them across the window or hand them to relays that will.
A wallet that never returns during the reveal window loses its vote.
That cost is bounded by the length of the reveal window, which is why
[Round Lifecycle] requires a deployment to publish it and choose it
with ordinary wallet usage in mind, and it is the reason the protocol
retains relays at all.

A secondary benefit is that the client's Merkle path is final once the
round enters REVEALING. A wallet syncs the round's tree once, after
voting closes, rather than maintaining a witness across the voting
window.

## Why the Chain Does Not Validate the Snapshot Roots

The complete remedy is for the vote chain to compute the snapshot roots
itself, so that they are consensus data rather than an input and no
verifier depends on administrators having performed
[Snapshot Derivation]. That requires every validator to follow Zcash
mainnet state, by running a Zcash node or trusting one, and to agree on
the value it reports: a change to the vote chain's consensus rules,
which this document does not make. The
roots are specified so that the change remains available. They are
deterministic functions of Zcash consensus state with an explicit
derivation procedure, so adding validation later means validators
implementing [Snapshot Derivation], not redefining the roots. Until
then a round's snapshot is only as correct as administrators' adherence
to [Round Attestation], and this document states that dependency rather
than leaving it implicit.



# Deployment

This ZIP does not specify a consensus change to the Zcash mainchain.

The parameters below are specific to a deployment rather than to the
protocol, but they are recorded here, with the protocol they
parameterise, rather than in a separate operational document. A
parameter stated apart from the claim that depends on it can drift from
it without either document becoming self-inconsistent, which is how
several of the divergences noted below arose.

A deployment MUST publish the values it uses for each parameter in this
section.

## Protocol parameters

| Parameter | Value | Constraint |
|---|---|---|
| $N_s$ | 16 | Shares per vote commitment. |
| $N_{\mathsf{opt}}$ | 8 | Option positions per share reveal; the maximum options per proposal. See [Vote Reveal Proof]. |
| Ballot unit | 12,500,000 zatoshi | 0.125 ZEC per ballot; see [Ballot Scaling]. |
| Share range | $[0, 2^{30})$ | Per-share plaintext bound. |
| Decomposition | Randomized | MUST satisfy [Vote Share]; even splitting is forbidden. |
| Shares per relay | 1 | See [Share Submission]. |
| $\Delta$ | 1 hour | Safety margin before $\mathsf{reveal}\_\mathsf{end}\_\mathsf{time}$; see [Submission Timing]. |
| $\mathsf{MAX}\_\mathsf{DELAY}$ | $W / 4$ | Delay draws above this are discarded and redrawn. |

## Round parameters

| Parameter | Why it is published |
|---|---|
| The decryption threshold $t$ and holder count $n$ | Bounds every amount-privacy claim in the protocol; see [Election Authority Key Ceremony]. |
| The organisation holding each election authority key share, and its registered Pallas key | Allows the role separation required in [Ratification] to be checked, and lets any party recompute the ceremony's verification keys. |
| The ceremony timeouts: commitment, dealing, complaint and acknowledgement | See [Election Authority Key Ceremony]. |
| The reveal window length, $\mathsf{reveal}\_\mathsf{end}\_\mathsf{time} - \mathsf{vote}\_\mathsf{end}\_\mathsf{time}$ | Bounds the period in which a wallet must return to reveal; see [Round Lifecycle]. |
| Zcash node implementations and versions relied on for $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ | That root is not Zcash consensus data, so which implementation served it is part of what a verifier checks; see [Snapshot Derivation]. |
| The administrators and their signing keys | Establishes whose attestations wallets recognise; see [Round Attestation]. |
| The administrator signature threshold $m$ | At least 2; see [Round Attestation]. |
| $\mathsf{min}\_\mathsf{confirmations}$ | The confirmation depth used when choosing the snapshot; see [Snapshot Configuration]. |
| The key-share retention period | Bounds the period over which amount-privacy claims hold; see [Election Authority Key Custody]. |

The RECOMMENDED value of $\mathsf{min}\_\mathsf{confirmations}$ is 100
blocks. The poll runner and each administrator SHOULD publish the
software and version they used to derive the snapshot roots, and the
values they derived, so that a disagreement about the snapshot can be
told apart from a disagreement about the derivation.

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

## Known divergences

The following differences between this specification and deployed
implementations are recorded so that they are not rediscovered as
defects. A deployment SHOULD resolve each, in the specification or in
the implementation.

- **Server-constructed reveal proofs.** Deployed implementations send
  the Vote Reveal Proof's witness material to a submission server,
  which constructs the proof. This ZIP requires the client to
  construct it and forbids sending that material to any party; see
  [Share Submission] and [Why Relays Do Not Construct Proofs].
- **Cleartext decisions.** Deployed implementations publish
  $\mathsf{vote}\_\mathsf{decision}$ in every share reveal. This ZIP
  makes it a private witness and publishes an option vector instead;
  see [Vote Reveal Proof].
- **Trusted dealer.** Deployed implementations generate the election
  authority key at a single dealer. This ZIP specifies distributed key
  generation; see [Election Authority Key Ceremony].
- **Overlapping reveal.** Deployed implementations accept share reveals
  during the voting window against any published root. This ZIP
  accepts them only during a reveal window after voting closes, and
  only against the final root; see [Round Lifecycle].
- **Last-moment window.** Earlier drafts defined a single-share window
  of $\min(0.1 \times \text{round duration}, 3600)$ seconds. Deployed
  implementations have used 40% of the round duration capped at six
  hours — for a 21-day round, the final six hours rather than the final
  hour. This ZIP removes single-share mode entirely; see
  [Why There Is No Single-Share Mode].
- **Per-server share limits.** Some deployed client libraries cap the
  number of a vote's shares sent to any one server at more than one.
  [Share Submission] now requires exactly one per relay.
- **Share decomposition.** Deployed implementations have divided the
  ballot count evenly across the $N_s$ shares. [Vote Share] forbids
  this; see [Why Randomized Share Decomposition].
- **Threshold.** The decryption threshold stated in companion documents
  and the threshold used in deployment have differed. The value in use
  MUST be published; see `draft-valargroup-shielded-voting-setup`
  [^voting-setup].


# Reference implementation

- [^ref-circuits] — Halo 2 circuits for the Delegation Proof, Vote
  Proof, and Vote Reveal Proof.
- [^ref-vote-sdk] — Cosmos SDK vote chain implementing the VCT,
  nullifier sets, encrypted share accumulator, ceremony, tally, and
  the submission server that this ZIP replaces with relays.
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
- **Key-share holder registration.** [Election Authority Key Ceremony]
  requires a key-share holder set, disjoint from the validator set,
  each with a registered Pallas public key. The operational
  specification (`draft-valargroup-shielded-voting-setup`
  [^voting-setup]) currently registers Pallas keys only for
  validators. How key-share holders are admitted, how they register
  keys, and how their addresses are recognised by the chain for
  ceremony and acknowledgement transactions is not yet specified.
- **Metadata linkage through relays.** [Share Submission] requires one
  relay per share and an independent network path per submission, and
  relies on the client to honour both. A relay operator that also
  operates the client's network path, or a coalition of relays pooling
  arrival logs with a coalition of $t$ key-share holders, could
  correlate by metadata what the protocol does not correlate by
  content. Role separation between relays and key-share holders is an
  operational requirement for `draft-valargroup-shielded-voting-setup`
  [^voting-setup].
- **Snapshot root validation by consensus**: see
  [Why the Chain Does Not Validate the Snapshot Roots].
- **Administrator keys**: wallets identify administrator keys as
  `draft-valargroup-shielded-voting-wallet-api` [^wallet-api]
  specifies, but how administrators are chosen, and how their keys are
  registered and rotated, is not specified.
- **Attestation timing**: an attestation covers
  $\mathsf{ea}\_\mathsf{pk}$, which exists only once the EA key ceremony
  completes, and the round opens as soon as it is ratified. Conforming
  wallets therefore cannot take part in a round until administrators
  attest to it after it opens, which shortens the effective voting window
  by however long that takes.
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
