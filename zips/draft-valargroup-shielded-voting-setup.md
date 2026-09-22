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
: A unique identifier for a voting round. See the "Poll Creation" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol] for the
  computation.

Poll runner
: The entity responsible for conducting a voting round.

Vote manager
: The on-chain role authorized to create voting rounds.

Administrator
: A party whose signature over a round's defining fields wallets
  recognise. See the "Round Attestation" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol].

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
  the "Election Authority Key Ceremony" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol] for how the keypair is generated and
  distributed, and the "Election Authority Key Custody" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol] for what the party
  generating it holds while it does so.

Key-share holder
: A holder of a share of a round's Election Authority private key.
  See the "Ratification" section of `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Snapshot height
: The Zcash mainnet block height at which eligible Orchard note balances
  are captured. See the "Snapshot Configuration" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol] for constraints.

For definitions of cryptographic terms including *alternate nullifier*,
*nullifier non-membership tree*, *nullifier domain*, *pool snapshot*, and
*claim*, see the Orchard Proof-of-Balance ZIP [^draft-balance-proof]. For
EA key ceremony terms, see the "Election Authority Key Ceremony" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol].


# Abstract

This ZIP specifies how to operate the infrastructure for Zcash shielded
coinholder voting: the operator roles, how validators and nullifier
services are provisioned and onboarded, how a deployment is organised so
that the parties who can group a voter's shares are not the parties who
can decrypt them, and the procedures by which any party audits a round.

The protocol itself — the cryptographic constructions, the delegation,
vote, reveal and tally phases, and the consensus rules a voting round
follows from creation through finalization — is specified in
`draft-valargroup-shielded-voting` [^draft-voting-protocol]. This ZIP
does not restate those rules; it states who carries them out and what
each of them must publish.


# Motivation

The Zcash Shielded Voting Protocol [^draft-voting-protocol] specifies
the protocol and the consensus rules of the vote chain. Running it
requires organisations to take on distinct roles, provision
infrastructure, and publish what they did, and the protocol's privacy
claims depend on how those roles are separated. This ZIP specifies that
operational layer.


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
  key ceremony. See the "Election Authority Key Ceremony" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol] and
  the "Election Authority Key Custody" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol].
- Validators holding sufficient stake to control block production can
  decline to include share reveal transactions. Because
  `vote_decision` appears in cleartext in every share reveal, selecting
  which votes to exclude by the option they support requires no
  decryption and no key material. See the "Transaction Inclusion" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol].
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
chain's voting round (see the "Poll Creation" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]).

The vote manager address is set in the genesis block (see
[Genesis Validator Setup]). The current vote manager MAY transfer
the role to another address via `MsgSetVoteManager`; the transfer
is atomic and moves the full account balance to the new address.
No other account can claim or reassign the role.

### Validator

Validators participate in consensus, the EA key ceremony (see
the "Election Authority Key Ceremony" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]), and automatic tally computation.
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

A nullifier service operator runs a server from which wallet clients
MAY obtain Merkle non-membership proofs against the snapshot's
nullifier non-membership tree, for clients that do not construct those
proofs themselves (see [Participation Flow]). The role is OPTIONAL: a
deployment that expects its clients to construct their own proofs need
not include one. Where a deployment does include one, the retrieval
protocol it offers is a property of that deployment and is not specified
here.

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

Where a deployment runs one, the service is run by a nullifier service
operator (see [Nullifier Service Operator]).

The service operates as a three-stage pipeline:

1. **Ingest**: fetch the Zcash mainnet nullifier set up to the
   chosen snapshot height from a Zcash node and persist it to
   local storage. The ingest pipeline MUST handle chain
   reorganisations as specified in the "Snapshot Derivation" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol].
2. **Export**: build the nullifier non-membership tree as specified
   in `draft-valargroup-orchard-balance-proof`
   [^draft-balance-proof] and export whatever query structures its
   retrieval protocol requires, so that the server can restart
   without rebuilding the tree from raw nullifiers.
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

The rules a voting round follows — its proposals and snapshot, its
creation, the attestation administrators make to it, its lifecycle
states, the election authority key ceremony, and the ratification by
key-share holders that opens voting — are consensus rules of the vote
chain, specified in the "Voting Round" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]. This section
states who carries out each step.

1. **Snapshot.** The poll runner chooses the snapshot block, reads both
   snapshot roots from a Zcash consensus node under its control, and
   coordinates with each nullifier service operator (see
   [Nullifier Service Operator]) so that their ingest and export
   pipelines (see [Nullifier Service]) have run to that height before
   the round opens.
2. **Creation.** The vote manager submits the round creation
   transaction. Where a chain is bootstrapped for the round, the roots
   are also passed into genesis state (see [Genesis Validator Setup]).
3. **Attestation.** Each administrator reads both roots from a Zcash
   consensus node under its own control, confirms they match the
   round, and publishes its attestation. Reading the roots from the
   proposer, or comparing bytes against the proposer's document, is not
   attestation.
4. **Ceremony and ratification.** Key-share holders take part in the
   election authority key ceremony and acknowledge their verified
   shares on chain; those acknowledgements ratify the round, and it
   opens once enough have been recorded.
5. **Tally.** After `vote_end_time`, key-share holders submit partial
   decryptions; the chain combines them and publishes the tally, or
   finalizes without one if too few arrive in time.

## Coinholder Participation

For a Zcash coinholder to participate in a voting round, they
must satisfy an eligibility precondition and follow a wallet-
driven participation flow. The protocol details for each step
are specified in companion ZIPs; this section ties them together
from the coinholder's perspective.

### Eligibility

Voting weight derives from the value held in a coinholder's
Orchard notes at the round's snapshot height (see
the "Snapshot Configuration" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]). Funds held in Sapling or transparent
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

1. **Obtain a non-membership proof.** Prove that the note's standard
   Orchard nullifier is absent from the set of nullifiers revealed at
   or before the snapshot height, against the snapshot's
   `nullifier_imt_root`. A client holding that nullifier set
   constructs the proof itself, as specified in
   `draft-valargroup-orchard-balance-proof` [^draft-balance-proof].
   A client that does not hold the set MAY obtain the proof from a
   nullifier service (see [Nullifier Service]); doing so reveals to
   that service which nullifier was asked about, and therefore which
   note, so a client SHOULD construct its own proof where it can.

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
  `draft-valargroup-orchard-balance-proof` [^draft-balance-proof].
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
Confirming that requires reading both roots from a Zcash consensus node
under the verifier's own control and comparing them with the round's,
by the procedure in the "Snapshot Derivation" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol], which also
establishes that the round is well formed. A verification of a round is
incomplete without it.

A round that auto-finalized due to a TALLYING timeout (see
the "Round Lifecycle" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]) verifies as having no tally;
this is itself a
verifiable property of the chain state.

Separately, the procedures above verify the transactions that are
present. They cannot establish that no transaction is missing; see
the "Transaction Inclusion" section of `draft-valargroup-shielded-voting` [^draft-voting-protocol].


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
| Stake distribution across validators | Determines the coalition size required to exclude transactions; see the "Transaction Inclusion" section of `draft-valargroup-shielded-voting` [^draft-voting-protocol]. |
| Software versions for the chain, circuits and client library | Required to reproduce or audit a round. |

The three operator sets MUST be disjoint. Round-level parameters — the
decryption threshold, administrator keys and threshold, confirmation
depth, and key custody — are published with the rules they parameterise,
in the "Deployment" section of `draft-valargroup-shielded-voting` [^draft-voting-protocol].


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
# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-1016]: [ZIP 1016: Community and Coinholder Funding Model](zip-1016.md)

[^draft-balance-proof]: [Draft ZIP: Orchard Proof-of-Balance](draft-valargroup-orchard-balance-proof.md)

[^draft-voting-protocol]: [Draft ZIP: Zcash Shielded Voting Protocol](draft-valargroup-shielded-voting.md)

[^draft-wallet-api]: [Draft ZIP: Shielded Voting Wallet API](draft-valargroup-shielded-voting-wallet-api.md)


[^draft-onchain-voting]: [Draft ZIP: On-chain Accountable Voting](draft-ecc-onchain-accountable-voting.md)

[^cosmos-staking]: [Cosmos SDK `x/staking` module documentation](https://docs.cosmos.network/main/build/modules/staking)

[^ref-vote-sdk]: [valargroup/vote-sdk: Cosmos SDK vote chain for shielded voting](https://github.com/valargroup/vote-sdk)

[^ref-nullifier-pir]: [valargroup/vote-nullifier-pir: PIR system for nullifier non-membership proofs](https://github.com/valargroup/vote-nullifier-pir)
