    ZIP: Unassigned
    Title: Zcash Shielded Coinholder Voting
    Owners: Dev Ojha <dojha@berkeley.edu>
            Roman Akhtariev <ackhtariev@gmail.com>
            Adam Tucker <adamleetucker@outlook.com>
            Greg Nagy <greg@dhamma.works>
    Status: Draft
    Category: Process
    Created: 2026-03-04
    License: MIT
    Pull-Request: <https://github.com/zcash/zips/pull/???>


# Terminology

The key words "MUST", "MUST NOT", "SHOULD", "RECOMMENDED" and "MAY" in
this document are to be interpreted as described in BCP 14 [^BCP14]
when, and only when, they appear in all capitals.

The terms below are to be interpreted as follows:

Vote chain
: The blockchain that serves as the single source of truth for voting
  operations. See [System Overview] for the state it maintains.

Voting round
: A complete instance of a coinholder vote, scoped to a single Zcash
  mainnet snapshot and a fresh Election Authority key.

Vote round ID
: A unique identifier for a voting round. See [Poll Creation] for the
  computation.

Poll runner
: The entity responsible for conducting a voting round.

Vote manager
: The on-chain role authorized to create voting rounds.

Administrator
: A party whose signature over a round's defining fields wallets
  recognise. See [Round Attestation].

Bootstrap operator
: The entity that provisions the vote chain genesis and initial
  validator set.

Validator
: A vote chain consensus participant. See [Validator] under Roles for
  responsibilities and keypair details.

Nullifier service operator
: The entity that runs the nullifier exclusion PIR server. See
  [Nullifier Service Operator] under Roles for responsibilities.

Bonded validator
: A validator whose stake is active under the standard Cosmos SDK
  `x/staking` module [^cosmos-staking]: its delegation is committed,
  it participates in consensus, and it is eligible to produce blocks.

Submission server
: An untrusted service that accepts encrypted vote share payloads
  from voters and submits the corresponding share reveal
  transactions to the vote chain. Share distribution, submission
  scheduling and the payload format are specified in
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Election Authority (EA)
: The El Gamal keypair under which a round's vote shares are encrypted,
  and whose private key decrypts the aggregate tally. A fresh keypair is
  generated for each round, and its private key is split into shares
  distributed to key-share holders, so that decrypting the tally requires
  a threshold of them acting together. See
  [Election Authority Key Ceremony] for how the keypair is generated and
  distributed, and [Election Authority Key Custody] for what the party
  generating it holds while it does so.

Key-share holder
: A holder of a share of a round's Election Authority private key.
  See [Ratification].

Snapshot height
: The Zcash mainnet block height at which eligible Orchard note balances
  are captured. See [Snapshot Configuration] for constraints.

For definitions of cryptographic terms including *alternate nullifier*,
*nullifier non-membership tree*, *nullifier domain*, *pool snapshot*, and
*claim*, see the Orchard Proof-of-Balance ZIP [^draft-balance-proof]. For
EA key ceremony terms, see [Election Authority Key Ceremony]. For
PIR-related terms, see
`draft-valargroup-nullifier-pir` [^draft-pir].


# Abstract

This ZIP specifies how to operate the infrastructure for Zcash shielded
coinholder voting. It defines a purpose-built vote chain built on Cosmos
SDK, three operator roles (bootstrap operator, vote manager, validator),
and the lifecycle of a voting round from snapshot selection through tally
verification.

A round is anchored to a specific Zcash mainnet block. This ZIP specifies
how the round's snapshot roots are derived from that block and
independently recomputed, how administrators attest to a round, and how
the holders of the Election Authority key ratify it before it opens.

The vote chain stores vote commitments in a Poseidon Merkle tree, tracks
three nullifier sets to prevent double-voting, and accumulates encrypted
vote shares as homomorphic El Gamal ciphertexts. Each transaction
(delegation, vote, share reveal) is verified by a zero-knowledge proof
on-chain. Validators join through an automated onboarding script that
handles binary distribution, key generation, and on-chain registration.
A separate nullifier service provides private information retrieval of
exclusion proofs so voters can prove note non-spending without revealing
which notes they hold.

Tally correctness is independently verifiable: validators submit partial
decryptions which are Lagrange-combined on-chain. Any party with access
to the chain state can re-derive the combination from the stored
partials and confirm the decrypted result.


# Motivation

The Zcash Shielded Voting Protocol [^draft-voting-protocol] defines a
cryptographic protocol for private coinholder voting. This ZIP specifies
the deployment infrastructure required to run that protocol: the vote
chain, operator roles, and voting round lifecycle.


# Privacy Implications

- Zero-knowledge proofs and encryption hide the contents of
  delegations, votes, and share reveals, but not the network-layer
  metadata associated with their submission. A vote chain validator
  sees the source IP, submission timestamp, connection correlation,
  and P2P propagation pattern of every transaction it receives.
- PIR queries against the nullifier service are private in content
  but not in timing: a nullifier service operator sees the source IP
  and time of each query, which reveals that a given client is
  participating in the current voting round.
- The vote chain is a public ledger. Transaction contents are
  encrypted or zero-knowledge-proven, but their existence, ordering,
  and block-inclusion timing are a permanent public record.
- Bootstrap operators learn the network identities of validators
  during onboarding (see [Onboarding Validators]).
- Validator power distribution affects the trust model for the EA
  key ceremony. See [Election Authority Key Ceremony] and
  [Election Authority Key Custody].
- Validators holding sufficient stake to control block production can
  decline to include share reveal transactions. Because
  `vote_decision` appears in cleartext in every share reveal, selecting
  which votes to exclude by the option they support requires no
  decryption and no key material. See [Transaction Inclusion].
- Where validators also hold election authority key shares, a coalition
  at the decryption threshold can additionally determine the weight of
  a vote before deciding whether to include it. This is one of the
  reasons the roles are separated in [Validator].


# Requirements

- A new poll runner can set up infrastructure and conduct a voting round
  by following this specification and the referenced companion ZIPs.
- A Zcash coinholder with eligible Orchard funds at the round's
  snapshot height can participate in the voting round using a
  conforming wallet client.
- The vote chain operates as a public, verifiable ledger — anyone can run
  a monitoring node to audit.
- The system operates with partial validator availability.
- The capabilities that validators hold over the outcome of a round,
  including the ability to exclude transactions, are documented rather
  than left implicit.
- No organisation operates more than one of the three roles that
  together would allow it to both group and decrypt a voter's shares.


# Non-requirements

- Governance policy decisions such as proposal eligibility, quorum
  requirements, and fund disbursement rules (see ZIP 1016 [^zip-1016]).


# Specification

## Proposals and Decisions

The voting process specified by this ZIP allows a poll runner to put
one or more questions — *proposals* — to eligible voters. For each
proposal, voters choose exactly one of a predefined set of labeled
*options*; the chosen option is the voter's *decision* for that
proposal.

### Structure of a proposal

Each proposal has:

- A **title**, short and human-readable.
- An optional **description** providing additional context.
- Between 2 and 8 **options**, each carrying a human-readable label.
  Option labels MUST be non-empty ASCII strings.

Proposals in a voting round are assigned 1-indexed sequential
identifiers; options within a proposal are assigned 0-indexed
sequential indices.

### Decisions

A **decision** is a voter's chosen option for a specific proposal,
represented as the option's 0-indexed position within that proposal's
option list. Decisions are recorded in the encrypted share
accumulator, keyed by `(proposal_id, vote_decision)`; see
`draft-valargroup-shielded-voting` [^draft-voting-protocol] for the
cryptographic construction.

### Kinds of polls that can be expressed

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

## System Overview

The coinholder voting system operates on a purpose-built Cosmos SDK vote
chain. Zcash mainnet snapshots provide the set of eligible Orchard note
balances.

The vote chain stores:

- A **Vote Commitment Tree** (VCT): a Poseidon Merkle tree of vote
  commitments.
- Three **nullifier sets**: governance nullifiers (alternate nullifiers
  from note claims), VAN nullifiers (from delegation consumption), and
  share nullifiers (from share reveals).
- An **encrypted share accumulator** per (proposal, decision): the
  homomorphic sum of El Gamal ciphertexts for each vote option.

The vote chain verifies a zero-knowledge proof for each transaction type:
delegation, vote, and share reveal. The proof circuits are specified in
`draft-valargroup-shielded-voting` [^draft-voting-protocol].

## Deployment Architecture

A complete deployment consists of:

- **Vote chain nodes** — one or more `svoted` instances running CometBFT
  consensus.
- **Submission servers** — untrusted services that accept vote share
  payloads from clients that cannot construct proofs locally, and
  submit the corresponding share reveal transactions on their behalf.
  These MUST be operated separately from the vote chain nodes and from
  the election authority key-share holders, and MUST NOT run in the
  `svoted` process; see [Validator] and [Why Roles Are Separated].
  Earlier deployments bundled the submission server into the `svoted`
  binary. Share distribution, submission scheduling and the payload
  format are specified in `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].
- **Nullifier service** — a PIR server that provides private nullifier
  exclusion proofs to voters (see [Nullifier Service]).
- **Vote configuration document** — a per-round document published
  by the vote manager that lists the network endpoints of the vote
  chain nodes and nullifier service operators participating in the
  round. The document format and distribution rules are specified
  in `draft-valargroup-shielded-voting-wallet-api`
  [^draft-wallet-api]; see also [Vote Configuration Publication].

## Roles

### Bootstrap Operator

The bootstrap operator generates the vote chain genesis block and
provisions the initial validator set. At genesis, a single vote
manager account is created with a balance of the chain's native token
(denom `usvote`) sized to fund all planned validators. From that
account the bootstrap operator funds each validator via
`MsgAuthorizedSend`, a transfer message gated by the vote chain's
ante handler: the vote manager MAY send to any address, and bonded
validators MAY send to the vote manager or to other bonded
validators; all other transfers — including the standard Cosmos bank
`MsgSend` and `MsgMultiSend` messages — are rejected.

The amount transferred to each validator at bonding time determines
their consensus voting power. An even distribution across validators
reduces the risk of consensus capture. The bootstrap operator's
activities are confined to genesis. The keypair that controls the
genesis `vote_manager` address continues afterwards as the initial
vote manager (see [Vote Manager]), and MAY transfer that role via
`MsgSetVoteManager`.

### Vote Manager

The vote manager is the only account authorized to initialize the
chain's voting round (see [Poll Creation]).

The vote manager address is set in the genesis block (see
[Genesis Validator Setup]). The current vote manager MAY transfer
the role to another address via `MsgSetVoteManager`; the transfer
is atomic and moves the full account balance to the new address.
No other account can claim or reassign the role.

### Validator

Validators participate in consensus, the EA key ceremony (see
[Election Authority Key Ceremony]), and automatic tally computation.
Each validator maintains three keypairs:

- **Consensus keypair**: used for CometBFT consensus.
- **Account keypair**: used for submitting chain transactions.
- **Pallas keypair**: used for ECIES key exchange during the EA ceremony.

Validators join the network by following the flow described in
[Onboarding Validators].

A validator MUST NOT also operate a submission server for the same
round, and MUST NOT also hold a share of the election authority key
for the same round.

These three roles — validating blocks, receiving voters' share
payloads, and holding key material that can decrypt those shares —
were previously performed by the same operators, on the reasoning that
introducing a separate operator class would add a trust assumption
without clear benefit. That reasoning is inverted: co-locating them
does not avoid an assumption, it conjoins two that must remain
independent. A submission server can determine which shares belong to
one voter, and a key share holder can decrypt them; a party holding
both roles needs no collusion to do both. See
[Why Roles Are Separated].

A deployment MUST publish which organisation operates each role for a
round, so that the independence of the three sets can be checked rather
than assumed.

### Nullifier Service Operator

A nullifier service operator runs the nullifier exclusion PIR
server that wallet clients query to obtain Merkle non-membership
proofs for the Zcash mainnet nullifier set at the snapshot
height. The PIR server is a separate binary, distributed
independently of the vote chain node, with its own ingest pipeline
and HTTP query endpoint. Specified in
`draft-valargroup-nullifier-pir` [^draft-pir].

## Vote Chain Infrastructure

### Genesis Validator Setup

The bootstrap operator obtains the vote chain binary `svoted`
either from a published release of the reference implementation
(see [Reference implementation]) or by building from source, and
initializes a new single-validator chain for the voting round.

Initialization proceeds as follows:

1. **Choose a chain identifier.** Each voting round uses its own
   chain identifier so that transactions signed for one round
   cannot be misinterpreted by software configured for another
   round.

2. **Generate the bootstrap operator's keypairs.** Generate a
   CometBFT consensus keypair, a Cosmos account keypair, and a
   Pallas keypair (see [Validator] for the role of each).

3. **Construct the genesis block.** Populate the genesis state
   with:
   - The chain identifier from step 1.
   - The vote manager singleton, set to an address controlled by
     the bootstrap operator (see [Bootstrap Operator]).
   - An initial balance for the vote manager account, in the
     chain's native token (`usvote`), sized to fund the planned
     validator set via subsequent authorized transfers.
   - A minimum ceremony validator count required before the EA
     key ceremony can proceed.
   - The bootstrap operator's own validator entry, bonded with
     consensus voting power and with its Pallas public key
     registered.
   - The standard Cosmos SDK module states (auth, bank, staking)
     as required by the Cosmos SDK runtime.

4. **Start the node.** Starting `svoted` begins block production
   and the chain transitions out of the genesis state.

The genesis state does not pre-populate voting rounds, nullifier
sets, tally results, or PIR data; these are all populated through
subsequent on-chain transactions during the voting round.

After the chain is producing blocks, the bootstrap operator
publishes the initial vote configuration document listing the
genesis node's public URL, so that joining validators and wallet
clients can find the network. See [Vote Configuration Publication].

### Onboarding Validators

A new validator joins the vote chain by following these steps:

1. **Acquire the vote chain binary.** Obtain `svoted` either from
   a published release of the reference implementation (see
   [Reference implementation]) or by building from source. Binary
   releases MUST be verified against a published checksum before
   use.

2. **Discover the network and initialize.** Read the vote
   configuration document for the target poll (see
   [Vote Configuration Publication]) to find at least one active
   validator. Fetch the chain's genesis block from that validator
   and initialize a local node directory.

3. **Generate keypairs.** Generate the validator's consensus
   keypair, account keypair, and Pallas keypair (see [Validator]).

4. **Sync the chain.** Start the node, connect to the active
   validators listed in the vote configuration document via
   CometBFT peer-exchange, and sync to the current height.

5. **Register in the vote configuration document.** Propose an
   update to the document adding the new validator's public URL.

6. **Wait for funding.** The vote manager reviews the proposed
   update, applies it, and funds the new validator's account via
   an authorized transfer (see [Bootstrap Operator] for the
   funding mechanism).

7. **Register on-chain.** Once funds are received, submit the
   validator registration transaction that wraps a standard
   Cosmos staking validator creation message together with the
   validator's Pallas public key, atomically binding the key to
   the new validator.

The amount transferred at step 6 determines the new validator's
consensus voting power. The vote chain rejects raw Cosmos staking
validator creation messages: every validator MUST be registered
through the wrapped form so that a Pallas public key is bound at
creation time. A validator without a registered Pallas key is
bonded for consensus but cannot participate in the EA ceremony
(see [Validator]).

The reference implementation (see [Reference implementation])
includes an automated `join.sh` script that performs the above
steps. The script is parameterized by the vote configuration
document URL; the same script is used for any poll and is not
specialized per poll.

### Nullifier Service

The nullifier service is an external service that lets voters
privately verify that their Orchard nullifiers are absent from
the Zcash mainnet nullifier set at the snapshot height, using
private information retrieval. A voter's PIR query reveals
neither which nullifier is being checked nor the answer to any
observer of the network.

The service is run by a nullifier service operator (see
[Nullifier Service Operator]) using the implementation referenced
in [Reference implementation]. The PIR construction and database
layout are specified in `draft-valargroup-nullifier-pir`
[^draft-pir].

The service operates as a three-stage pipeline:

1. **Ingest**: fetch the Zcash mainnet nullifier set up to the
   chosen snapshot height from a Zcash node and persist it to
   local storage.
2. **Export**: build the nullifier non-membership tree (an Indexed
   Merkle Tree as specified in
   `draft-valargroup-orchard-balance-proof` [^draft-balance-proof])
   and export the PIR database tiers as specified in
   `draft-valargroup-nullifier-pir` [^draft-pir]. The exported
   files allow the server to restart without rebuilding the tree
   from raw nullifiers.
3. **Serve**: accept PIR queries from voters and return encrypted
   responses over HTTP. The operator publishes the service URL by
   adding it to the vote configuration document (see
   [Vote Configuration Publication]).

The PIR client-server query and response wire format is not yet
specified in any normative document. See [Open Issues].

### Vote Configuration Publication

The vote manager publishes a vote configuration document for the
chain's voting round, in the format and via the publication
channel described in `draft-valargroup-shielded-voting-wallet-api`
[^draft-wallet-api]. Wallets and joining validators fetch the
document from that channel; the vote chain itself does not
provide a service-discovery endpoint.

The vote manager publishes the initial document once the genesis
validator is producing blocks. New validators and nullifier service
operators are added by proposing updates to the document, which the
vote manager reviews and applies; for validators this proposal is
automated by the join.sh flow described in [Onboarding Validators],
while nullifier service operators propose their entries directly.
The document is incremental — new entries are added without
removing earlier ones — and any active entry can serve as an entry
point for new joiners; there is no distinguished seed node.

Each voting round runs on its own vote chain with its own
publication channel. The vote manager announces the channel URL
out-of-band before the round opens, so that wallet implementers
can configure their wallets to fetch from it — either as a
bundled default or as a user-supplied setting. There is no
automated cross-poll discovery; a new poll requires the vote
manager to communicate the new channel to wallets and end users
through whatever distribution path they choose.

## Conducting a Voting Round

### Snapshot Configuration

A voting round is anchored to a Zcash mainnet snapshot: a single
mainnet block, identified by both its height $H$ (the snapshot height)
and its hash $\mathsf{snapshot}\_\mathsf{blockhash}$, at which the
eligible Orchard pool is captured. The poll runner chooses the block
subject to the following constraints:

- $H$ MUST be at or after NU5 activation, since the protocol requires
  Orchard.
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

1. **Determine the snapshot roots.** The Orchard note commitment tree
   root ($\mathsf{nc}\_\mathsf{root}$) and the nullifier non-membership
   tree root ($\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$) are
   the two roots $\mathsf{rt^{cm}}$ and $\mathsf{rt^{excl}}$ of the pool
   snapshot at height $H$, as defined in the "Pool Snapshot" section of
   `draft-valargroup-orchard-balance-proof` [^draft-balance-proof], on
   the chain whose block at height $H$ has hash
   $\mathsf{snapshot}\_\mathsf{blockhash}$. No party has discretion over
   their values. The poll runner derives them by the procedure in
   [Snapshot Derivation].

2. **Ensure the nullifier service has the snapshot's PIR
   database.** The poll runner coordinates with each nullifier
   service operator (see [Nullifier Service Operator]) so that
   their ingest and export pipelines (see [Nullifier Service])
   have run to the chosen height before the round opens, so
   wallets can query exclusion proofs against that snapshot.

3. **Use the values during chain bootstrap.**
   $\mathsf{nc}\_\mathsf{root}$ and
   $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ are passed
   into the genesis state (see [Genesis Validator Setup]) and
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
2. Obtain $\mathsf{nc}\_\mathsf{root}$: the Orchard note commitment
   tree root as of the end of block $H$. This is Zcash consensus data.
   A node computes it while validating the chain and exposes it as the
   Orchard anchor at that height.
3. Obtain $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$: the root
   of the nullifier non-membership tree over every Orchard nullifier
   revealed at or before $H$, constructed as specified in
   `draft-valargroup-orchard-balance-proof` [^draft-balance-proof].
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
[^draft-wallet-api], and derives no roots of its own.

**Node requirements.** A deployment MUST identify the Zcash node
implementations and versions it relies on to serve
$\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ (see [Deployment]).
Because that root is not consensus data, two node implementations can
disagree about it with no Zcash consensus rule to settle the
disagreement. The construction specified in
`draft-valargroup-orchard-balance-proof` [^draft-balance-proof] is the
sole authority: a party that obtains differing roots for the same
$(H, \mathsf{snapshot}\_\mathsf{blockhash})$ from independent nodes MUST
treat the round as not well formed until the discrepancy is resolved.

The served root is correct only if the tree behind it covers every
Orchard nullifier revealed at or before $H$ and nothing else. A node
maintaining that index incrementally MUST handle chain reorganisations
as specified for the nullifier service's ingest pipeline in
`draft-valargroup-nullifier-pir` [^draft-pir], and MUST NOT serve a root
for $H$ until it has confirmed that the block it ingested at $H$ has
hash $\mathsf{snapshot}\_\mathsf{blockhash}$.

### Poll Creation

The vote manager initializes the chain's voting round by submitting
a transaction carrying the client-supplied subset of the `VoteRound`
structure specified in `draft-valargroup-shielded-voting-wallet-api`
[^draft-wallet-api]. The vote manager supplies `snapshot_height`,
`snapshot_blockhash`, `proposals_hash`, `vote_end_time`,
`nullifier_imt_root`, `nc_root`, `proposals`, `title`, and
`description`; the transaction's signer becomes the `creator` field
of the resulting `VoteRound`. The chain derives the remaining
fields (`vote_round_id`, `status`, `ea_pk`, `created_at_height`)
at inclusion or during the round lifecycle.

The chain rejects the transaction if the signer is not the current
vote manager, or if the `proposals` field violates the constraints
in [Proposals and Decisions].

The vote chain derives the 32-byte `vote_round_id` from the
transaction fields after inclusion, using the Poseidon construction
specified in the "Voting Round Identifier" section of
`draft-valargroup-shielded-voting`
[^draft-voting-protocol-vri]. The result is a Pallas field element
so that the round ID can enter ZKP circuits as a public input.

The round enters the **PENDING** state. The EA key ceremony (see
[Election Authority Key Ceremony]) runs automatically. Once it completes and at least $t$ key-share holders have
ratified the round (see [Ratification]), the round transitions to
**ACTIVE**, the voting window opens, and the transition timestamp is
recorded as `ceremony_phase_start`. Clients use
`ceremony_phase_start` together with `vote_end_time` to construct their
share submission schedule, as specified in the "Submission Timing"
section of `draft-valargroup-shielded-voting` [^draft-voting-protocol].
There is no last-moment buffer: the single-share mode that earlier
drafts defined for the end of the voting window has been removed, and a
client near the deadline compresses its schedule rather than
concentrating its weight.

### Round Attestation

Administrators attest to a round by signing its defining fields. A
wallet accepts a round only with at least $m$ valid attestations from
administrators it recognises, as specified in
`draft-valargroup-shielded-voting-wallet-api` [^draft-wallet-api], and
the administrator signature threshold $m$ MUST be at least 2.

The bytes covered by an attestation are the concatenation, in this
order, of:

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
2. **ACTIVE**: ceremony complete and round ratified, voting window
   open. Voters may delegate, vote, and submit shares (see
   `draft-valargroup-shielded-voting`
   [^draft-voting-protocol]).
3. **TALLYING**: `vote_end_time` has passed. Validators submit
   partial decryptions, the chain combines them, and tally
   decryption runs automatically (see the "Tally" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol]). The
   chain enforces a bounded timeout on the TALLYING state: if a
   tally is not submitted within this timeout, the round
   auto-finalizes with no tally, preserving liveness.
4. **FINALIZED**: tally published and verifiable. A round that
   auto-finalized due to a TALLYING timeout publishes no tally.

### Election Authority Key Ceremony

Each round uses a fresh election authority keypair
$(\mathsf{ea}\_\mathsf{sk}, \mathsf{ea}\_\mathsf{pk})$. The ceremony
that produces it runs automatically when the round enters PENDING.

Scoping the key to one round bounds the damage from a key compromise to
that round, and means a key-share holder that leaves the validator set
cannot decrypt later rounds. The cryptographic constructions used below
— El Gamal on Pallas, ECIES on Pallas, and Chaum-Pedersen DLEQ proofs —
are specified in `draft-valargroup-shielded-voting`
[^draft-voting-protocol].

**Eligibility.** Every validator holding a registered Pallas public key
at the time of round creation is eligible. Registration is bound at
validator creation, as specified in [Onboarding Validators]; a validator
without a registered Pallas key is bonded for consensus but takes no
part in the ceremony.

**Dealer selection.** The next block proposer acts as dealer.

**Key generation and distribution.** Let $n$ be the number of eligible
validators and let $t = \lceil n/2 \rceil + 1$, with a minimum of 2, be
the round's decryption threshold. The dealer:

1. Samples $\mathsf{ea}\_\mathsf{sk}$ uniformly at random and computes
   $\mathsf{ea}\_\mathsf{pk}$.
2. Constructs a random polynomial $f$ of degree $t - 1$ over the Pallas
   scalar field with $f(0) = \mathsf{ea}\_\mathsf{sk}$.
3. Evaluates $f(i)$ for $i = 1, \ldots, n$ to produce one Shamir share
   per eligible validator.
4. Encrypts each share $f(i)$ to that validator's registered Pallas key
   using ECIES, with a fresh ephemeral scalar per recipient.
5. Publishes to the chain: $\mathsf{ea}\_\mathsf{pk}$, the threshold
   $t$, every encrypted share, and every verification key
   $\mathsf{VK}_i = [f(i)]\, G$.
6. Securely erases $\mathsf{ea}\_\mathsf{sk}$, the polynomial
   coefficients, and every share value.

Step 6 is not verifiable by any other party.
[Election Authority Key Custody] states what follows from that, and
what a deployment MUST do about it.

**Acknowledgement.** Each eligible validator decrypts its share,
verifies that $[f(i)]\, G$ equals the published $\mathsf{VK}_i$,
stores the share, and submits an acknowledgement transaction carrying

$$\mathsf{SHA256}\bigl(\texttt{"ack"} \mathbin\| \mathsf{vote}\_\mathsf{round}\_\mathsf{id} \mathbin\| \mathsf{ea}\_\mathsf{pk} \mathbin\| \mathsf{validator}\_\mathsf{address}\bigr)$$

A validator MUST NOT acknowledge a share that fails the verification
key check. Committing to $\mathsf{ea}\_\mathsf{pk}$ keeps an
acknowledgement from carrying over to a round rekeyed after the fact;
committing to `vote_round_id` keeps it from carrying over to another
round under the same key. This acknowledgement is also the holder's
ratification of the round; see [Ratification].

**Confirmation.** The ceremony confirms when every eligible validator
has acknowledged, or, after the acknowledgement timeout, when at least
$t$ have. Validators that did not acknowledge are dropped from the
round and increment a consecutive-miss counter; after three consecutive
misses a validator MUST be jailed, which removes it from the active set
without burning bonded value. Ceremony non-participation is a liveness
failure, not a safety violation, which is why it is penalised by
jailing rather than by slashing.

If fewer than $t$ eligible validators acknowledge within the timeout,
the ceremony resets and a new dealer is selected.

Requiring $t$ acknowledgements before confirmation is deliberate: the
number of acknowledgements required to confirm is the same $t$ used for
threshold decryption. Were confirmation to require fewer, a round could
open that could never be tallied.

**Validator set changes.** A validator joining during an active round
receives no share for that round and MUST wait for the round to
complete. A validator leaving retains its share and cannot be compelled
to delete it; per-round keys bound what that share is worth, since it
opens nothing in any other round.

**Timing parameters.** A deployment MUST publish the ceremony deal
timeout and the acknowledgement timeout it applies, and the
consecutive-miss count at which a validator is jailed.

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

**Share generation.** A ceremony that generates the key at a single
party and distributes shares from it — a trusted dealer, as specified
in [Election Authority Key Ceremony] — gives that party
the full Election Authority private key for the duration of the
ceremony. The claim that no single party holds it holds only after the
ceremony completes, and only if the dealer destroyed its copy, which no
other party can verify. A deployment using a trusted dealer MUST
identify the party that acted as dealer for each round, and SHOULD adopt
distributed key generation, or publish verifiable secret sharing
commitments, so that key-share holders can confirm their shares are
consistent with $\mathsf{ea}\_\mathsf{pk}$ without trusting the dealer.

**Retention.** Each key-share holder MUST destroy its share once the
round is finalized and its tally published, and a deployment MUST
publish the retention period it applies. The encrypted shares of every
individual vote remain on the vote chain permanently, and their
encryption is not post-quantum, so retained shares are a live capability
against a permanent record of individual voters' balances, not a dormant
convenience. Retention is not needed for audit: the partial decryptions
and their DLEQ proofs are published on chain and can be re-verified at
any time without the key. A deployment that retains shares nonetheless
MUST state for how long, and MUST treat that period as the period over
which its amount-privacy claims hold.

## Coinholder Participation

For a Zcash coinholder to participate in a voting round, they
must satisfy an eligibility precondition and follow a wallet-
driven participation flow. The protocol details for each step
are specified in companion ZIPs; this section ties them together
from the coinholder's perspective.

### Eligibility

Voting weight derives from the value held in a coinholder's
Orchard notes at the round's snapshot height (see
[Snapshot Configuration]). Funds held in Sapling or transparent
pools at the snapshot height carry no voting weight; coinholders
who wish to participate with such funds MUST migrate them to
Orchard before the snapshot height.

### Wallet Setup

A wallet client obtains the vote configuration document for the
round (see [Vote Configuration Publication]), which lists the
vote chain endpoints, the nullifier service endpoints, and the
protocol versions in use. The configuration document schema and
the wallet-side validation rules are specified in
`draft-valargroup-shielded-voting-wallet-api`
[^draft-wallet-api].

### Participation Flow

For each Orchard note the coinholder uses as voting weight, the
wallet performs:

1. **Retrieve a non-membership proof.** Query a nullifier service
   endpoint to retrieve a Merkle non-membership proof for the
   note's alternate nullifier against the snapshot's
   `nullifier_imt_root`, as specified in
   `draft-valargroup-nullifier-pir` [^draft-pir].

2. **Submit a delegation transaction.** Construct a Delegation
   Proof asserting ownership of the eligible note without
   revealing which note it is, and submit the transaction to a
   vote chain endpoint listed in the vote configuration document.
   This produces a Vote Authority Note (VAN) on the vote
   commitment tree. The proof construction and the on-chain
   handling are specified in the Delegation Phase of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol].

For each proposal the coinholder votes on, the wallet performs:

3. **Submit a vote transaction.** Construct a Vote Proof
   consuming the current VAN and producing a new VAN with the
   relevant proposal authority bit cleared, plus a Vote
   Commitment binding the chosen option, as specified in the
   Vote Phase of `draft-valargroup-shielded-voting`
   [^draft-voting-protocol].

4. **Submit encrypted vote shares.** Send the share payloads to
   submission server endpoints, which queue them and submit
   share reveal transactions on the coinholder's behalf at
   client-specified times. Share decomposition, server selection and
   submission scheduling are specified in
   `draft-valargroup-shielded-voting` [^draft-voting-protocol].

After `vote_end_time`, the coinholder may verify the final tally
following [Verification and Auditing].

## Transaction Inclusion

The vote chain is a CometBFT chain, and its validators determine which
transactions enter blocks. This section states the consequences for a
voting round, which are not otherwise recorded in this or any companion
specification.

**What validators can do.** Validators controlling enough stake to
control block production can decline to include share reveal
transactions. Because $\mathsf{vote}\_\mathsf{decision}$ appears in
cleartext in every share reveal transaction, and running per-option
counts are public while a round is open, selecting which transactions
to exclude according to the option they support requires no decryption,
no key material, and no cooperation from any other party.

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
specified; see [Open Issues].

**Detection.** Exclusion is detectable but not provable from chain
state alone. An excluded transaction leaves no record on the chain that
excluded it. Available signals are: a count of votes cast against the
count of shares revealed; the contents of honest nodes' mempools,
compared with what was subsequently included; and voters observing
that their own shares never appeared. The last is currently unavailable
in practice, because a voter querying the chain for their own share
nullifiers reveals which nullifiers are theirs; see
[^draft-voting-protocol].

A deployment SHOULD publish, for each round, the count of share reveal
transactions accepted into the mempool alongside the count included in
blocks, from more than one operator, so that a discrepancy is visible
without requiring any party to be trusted.

**Threshold.** A deployment MUST publish the stake distribution across
validators for a round, and the proportion of stake required to control
block production, so that the size of the coalition required to exclude
transactions is a published figure rather than an inferred one.


## Verification and Auditing

The vote chain is publicly readable. Any party running a full
node of the chain — a validator, the vote manager, or an
independent observer — can verify all aspects of a voting round
by replaying chain state and applying the verification
procedures defined in companion ZIPs.

This section uses the following procedures imported by
reference:

- **Out-of-Circuit Verification of the Claim proof** in
  `draft-valargroup-orchard-balance-proof`
  [^draft-balance-proof] — the abstract balance-proof primitive
  on which the Delegation Proof is built.
- **Out-of-Circuit Verification of the Delegation Proof, the
  Vote Proof, and the Vote Reveal Proof** in
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].
- **Authentication Path** verification and the exclusion-range
  check for the nullifier non-membership tree, in
  `draft-valargroup-nullifier-pir` [^draft-pir].
- **Proof Verification** (DLEQ verification of partial
  decryptions) and the **Tally** procedure (Lagrange combination
  and plaintext recovery), in `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].

A full-node operator combines these procedures to verify a
voting round across three layers:

- **Per-transaction zero-knowledge proof verification.** Each
  delegation, vote, and share reveal transaction carries a Halo
  2 proof that is checked at inclusion against the
  Out-of-Circuit Verification rules in
  `draft-valargroup-shielded-voting`. Those rules apply the
  Claim proof verification from the balance proof ZIP and the
  Authentication Path / exclusion-range checks from the PIR
  ZIP transitively, so a single proof verification enforces the
  structural correctness of all three layers at once.

- **Nullifier set integrity.** The chain maintains three
  nullifier sets — governance nullifiers (from delegation), VAN
  nullifiers (from voting), and share nullifiers (from share
  reveals) — and rejects any transaction that would re-use a
  previously published nullifier. A full-node operator confirms
  set integrity by replaying every accepted transaction and
  checking that no nullifier appears twice.

- **Tally correctness.** A full-node operator re-aggregates the
  encrypted share ciphertexts per (`proposal_id`, `vote_decision`)
  from the on-chain share reveal transactions, applies the DLEQ
  Proof Verification to each stored partial decryption,
  re-derives the Lagrange combination, and confirms the
  decrypted aggregate following the Tally procedure in
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

The three layers above verify that the transactions in a round are
well formed and correctly accumulated *with respect to* the round's
snapshot roots. They do not establish that those roots are correct, and
the roots are not derived from vote chain state: they are supplied as
input at round creation and are not validated by consensus.

A full-node operator therefore does not need to trust another
participant for the three layers above, but does depend on the
snapshot roots being correct, which this chain does not establish.
Confirming that requires recomputing both roots from Zcash mainnet
state by the procedure in [Snapshot Derivation], which also
establishes that the round is well formed. A verification of a round is
incomplete without it.

A round that auto-finalized due to a TALLYING timeout (see
[Round Lifecycle]) verifies as having no tally; this is itself a
verifiable property of the chain state.

Separately, the procedures above verify the transactions that are
present. They cannot establish that no transaction is missing; see
[Transaction Inclusion].


# Rationale

## Why Roles Are Separated

Three roles in this system have to be held by different organisations:
the validator that decides which transactions enter blocks, the
submission server that receives voters' share payloads, and the holder
of a share of the election authority key.

Earlier drafts bundled all three into one binary run by one operator
set, reasoning that validators already participate in the key ceremony,
so a separate submission server operator class would add a trust
assumption without clear benefit.

The reasoning treats a trust assumption as a cost to be minimised by
reducing the number of distinct parties. What matters is not how many
parties there are, but which capabilities land together. A submission
server can determine which shares belong to one voter, because the
payloads it receives carry values common to all shares of a vote. A key
share holder can, with enough peers, decrypt those shares. Separately,
neither recovers a voter's balance. Held by the same organisation, they
do, with no collusion required — and where the same organisation also
validates, it can act on what it learns by deciding what to include.

Separating the roles does not add an assumption. It restores one that
bundling had quietly removed, by making a coalition necessary where
previously a single operator sufficed. It also makes the assumption
checkable: with the operator of each role published, anyone can verify
that the three sets are disjoint, which is not possible when one
binary performs all three.

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

## Other Design Choices

**Separate vote chain (not Zcash mainnet)**: the vote chain is purpose-built
for governance with ZKP-optimized state transitions (Poseidon hashing, custom
transaction types). Zcash mainnet's transaction throughput and scripting model
are not designed for interactive multi-phase voting protocols.

**Orchard-only snapshots**: the voting protocol is built on Orchard's
circuit-friendly primitives (Poseidon hashing, Pallas curve). Sapling
and transparent pools use incompatible cryptographic constructions.
The corresponding requirement on coinholders is stated in
[Eligibility].

**Cosmos SDK**: provides a mature BFT consensus engine (CometBFT),
validator lifecycle management (bonding, jailing for missed blocks or
missed ceremony acknowledgements, consensus power distribution), and a
transaction pipeline that can be extended with custom message types and
ante handlers for ZKP verification. The alternative — building a chain
from scratch — would duplicate well-tested consensus infrastructure.

**Funding equals voting power**: bonding serves three purposes: it
determines which validators participate in consensus and the EA key
ceremony (and thus can decrypt the tally), it enables jailing of
inactive validators who miss blocks or ceremony acknowledgements, and
an even funding split gives each validator a roughly equal probability
of becoming the block proposer — not important for correctness, but
important for liveness.

**Automated validator onboarding**: `join.sh` eliminates manual
coordination between the bootstrap operator and joining validators. The
self-registration, admin-approval, and auto-bonding flow allows the
network to grow without requiring validators to build from source or
understand Cosmos SDK tooling.

**Vote manager reassignment**: only the current vote manager can
transfer the role (via `MsgSetVoteManager`). This is a deliberate
single-party control: the vote manager can create rounds but cannot
forge votes, and the worst-case mitigation for a compromised vote
manager is to spin up a new chain.


# Deployment

A deployment MUST publish the following for each round. These are
recorded here, alongside the properties that depend on them, so that a
divergence between a published value and the specification is visible
to a reader of this document.

| Parameter | Why it is published |
|---|---|
| The organisation operating each validator | Establishes the validator set for the round. |
| The organisation operating each submission server | Allows the role separation required in [Validator] to be checked. |
| The organisation holding each election authority key share | As above. |
| Stake distribution across validators | Determines the coalition size required to exclude transactions; see [Transaction Inclusion]. |
| The decryption threshold $t$ and holder count $n$ | Bounds every amount-privacy claim in the protocol. |
| Software versions for the chain, circuits and client library | Required to reproduce or audit a round. |
| Zcash node implementations and versions relied on for $\mathsf{nullifier}\_\mathsf{imt}\_\mathsf{root}$ | That root is not Zcash consensus data, so which implementation served it is part of what a verifier checks; see [Snapshot Derivation]. |
| The administrators and their signing keys | Establishes whose attestations wallets recognise; see [Round Attestation]. |
| The administrator signature threshold $m$ | At least 2; see [Round Attestation]. |
| $\mathsf{min}\_\mathsf{confirmations}$ | The confirmation depth used when choosing the snapshot; see [Snapshot Configuration]. |
| The party that dealt the Election Authority key, if a trusted dealer was used | See [Election Authority Key Custody]. |
| The key-share retention period | Bounds the period over which amount-privacy claims hold; see [Election Authority Key Custody]. |

The three operator sets MUST be disjoint. A deployment that cannot
satisfy this MUST publish which roles are co-located and which
organisations hold them, so that the resulting capability is a
disclosed property of that deployment rather than an unstated one.

The RECOMMENDED value of $\mathsf{min}\_\mathsf{confirmations}$ is 100
blocks. The poll runner and each administrator SHOULD publish the
software and version they used to derive the snapshot roots, and the
values they derived, so that a disagreement about the snapshot can be
told apart from a disagreement about the derivation.


# Reference implementation

- [^ref-vote-sdk] — Cosmos SDK vote chain (`svoted`) implementing
  the chain-side state, ceremony, tally, and submission server.
- [^ref-nullifier-pir] — PIR server and client for privately
  retrieving nullifier non-membership proofs.


# Open Issues

- **Role consolidation**: evaluate whether the bootstrap operator and
  vote manager concepts (already the same keypair at genesis; see
  [Bootstrap Operator]) merit separate treatment in the specification.
- **PIR client-server wire format**: the query and response wire
  format used between wallet clients and the nullifier service
  (see [Nullifier Service]) is not currently specified in any
  ZIP. The PIR draft scopes its outer transport out of remit, and
  the wallet API ZIP describes only the endpoint URLs. A normative
  spec home is needed before wallet clients and nullifier service
  servers from independent implementations can interoperate.
- **Snapshot root validation by consensus**: see
  [Why the Chain Does Not Validate the Snapshot Roots].
- **Implementation diversity**: a result is described as coinholder
  sentiment, but where one client is the only practical way to vote, its
  defaults — how it splits shares, which servers it uses, when it
  submits — are the protocol as every voter experiences it. The
  conditions under which a result may be described as representative are
  not specified. Requiring that some number of independent
  implementations be *available* is not checkable, and is met by two
  implementations one of which casts nearly every ballot. A checkable
  form would bound the share of ballots, or of voting weight, cast
  through any one implementation, and would bind the deployment that
  publishes the result.
- **Trusted dealer**: [Election Authority Key Ceremony] specifies a
  single dealer that holds $\mathsf{ea}\_\mathsf{sk}$ for the duration of
  key generation and is trusted to erase it. Distributed key generation,
  or at minimum published verifiable secret sharing commitments, would
  remove that trust; [Election Authority Key Custody] requires a
  deployment to name the dealer and recommends the upgrade, but neither
  is specified here.
- **Administrator keys**: wallets identify administrator keys as
  `draft-valargroup-shielded-voting-wallet-api` [^draft-wallet-api]
  specifies, but how administrators are chosen, and how their keys are
  registered and rotated, is not specified.
- **Attestation timing**: an attestation covers
  $\mathsf{ea}\_\mathsf{pk}$, which exists only once the EA key ceremony
  completes, and the round opens as soon as it is ratified. Conforming
  wallets therefore cannot take part in a round until administrators
  attest to it after it opens, which shortens the effective voting window
  by however long that takes.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-1016]: [ZIP 1016: Community and Coinholder Funding Model](zip-1016.md)

[^draft-balance-proof]: [Draft ZIP: Orchard Proof-of-Balance](draft-valargroup-orchard-balance-proof.md)

[^draft-voting-protocol]: [Draft ZIP: Zcash Shielded Voting Protocol](draft-valargroup-shielded-voting.md)

[^draft-voting-protocol-vri]: [Draft ZIP: Zcash Shielded Voting Protocol, Section: Voting Round Identifier](draft-valargroup-shielded-voting.md#voting-round-identifier)

[^draft-pir]: [Draft ZIP: Private Information Retrieval for Nullifier Exclusion Proofs](draft-valargroup-nullifier-pir.md)


[^draft-wallet-api]: [Draft ZIP: Shielded Voting Wallet API](draft-valargroup-shielded-voting-wallet-api.md)


[^draft-onchain-voting]: [Draft ZIP: On-chain Accountable Voting](draft-ecc-onchain-accountable-voting.md)

[^cosmos-staking]: [Cosmos SDK `x/staking` module documentation](https://docs.cosmos.network/main/build/modules/staking)

[^ref-vote-sdk]: [valargroup/vote-sdk: Cosmos SDK vote chain for shielded voting](https://github.com/valargroup/vote-sdk)

[^ref-nullifier-pir]: [valargroup/vote-nullifier-pir: PIR system for nullifier non-membership proofs](https://github.com/valargroup/vote-nullifier-pir)
