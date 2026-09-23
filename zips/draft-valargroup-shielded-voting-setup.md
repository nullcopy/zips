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

Relay
: An untrusted store-and-forward service to which a voter's client MAY
  hand a finished share reveal message, together with the time at which
  to submit it, for later submission to the vote chain. A relay
  constructs no proofs and receives no witness material. See
  [Relay Operator] under Roles, and the "Share Submission" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Relay operator
: The entity that runs one or more relays. See [Relay Operator] under
  Roles for responsibilities.

Ironwood pool
: The Zcash shielded pool over which votes are weighted. The Ironwood
  pool uses the Orchard protocol: its notes, key hierarchy, note
  commitment tree, nullifiers, signatures and proving system are those
  of Orchard. References in this ZIP to Orchard keys, notes, nullifiers
  or circuits refer to those constructions as used in the Ironwood pool.

Election Authority (EA)
: The El Gamal keypair under which a round's vote shares are encrypted,
  and whose private key decrypts the aggregate tally. A fresh keypair is
  generated for each round by distributed key generation among the
  round's key-share holders, so that no party ever holds the private
  key and decrypting the tally requires a threshold of holders acting
  together. See the "Election Authority Key Ceremony" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol] for the
  ceremony, and the "Election Authority Key Custody" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol] for what
  each holder retains and for how long.

Key-share holder
: An organisation admitted to a round's holder set, which holds a share
  of the round's Election Authority private key produced by the key
  ceremony. Not a validator or relay operator of the same round. See
  [Key-Share Holder] under Roles, and the "Ratification" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Snapshot height
: The Zcash mainnet block height at which eligible Ironwood pool
  balances are captured. See the "Snapshot Configuration" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol] for constraints.

For definitions of cryptographic terms including *alternate nullifier*,
*nullifier non-membership tree*, *nullifier domain*, *pool snapshot*, and
*claim*, see the Orchard Proof-of-Balance ZIP [^draft-balance-proof]. For
EA key ceremony terms, see the "Election Authority Key Ceremony" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol].


# Abstract

This ZIP specifies how to operate the infrastructure for Zcash shielded
coinholder voting: the operator roles, how validators, key-share
holders, relays and nullifier services are provisioned and onboarded,
how a deployment is organised so that no organisation both sees a
voter's share reveals arrive, or decides which of them enter blocks, and
holds key material that can decrypt them, and the procedures by which
any party audits a round.

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
- A relay operator learns, from each share reveal message handed to
  it, what a chain observer learns from the same message once it is
  recorded, plus the network origin of the client that handed it over
  and the requested submission time. See [Relay Operator] and the
  "Privacy Implications" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].
- The composition of the key-share holder set, and the decryption
  threshold $t$ the ceremony fixes from its size, bound every
  amount-privacy claim in the protocol. See the "Election Authority
  Key Ceremony" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol] and the "Election Authority Key Custody"
  section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].
- Validators holding sufficient stake to control block production can
  decline to include share reveal transactions. A share reveal exposes
  its proposal identifier but not its decision, and per-option totals
  are not public while a round is open, so validators acting alone
  cannot select which reveals to exclude by the option they support;
  they can exclude by proposal, by time of arrival, by network origin,
  or wholesale. See the "Transaction Inclusion" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].
- A coalition of validators and $t$ key-share holders could decrypt
  reveals as they arrive and exclude them by option and by weight. A
  coalition of relay operators and $t$ key-share holders could group a
  voter's shares by the metadata relays see and decrypt them. These
  are the reasons the three operator sets are required to be disjoint
  in [Role Separation].


# Requirements

- A new poll runner can set up infrastructure and conduct a voting round
  by following this specification and the referenced companion ZIPs.
- A Zcash coinholder with eligible Ironwood pool funds at the round's
  snapshot height can participate in the voting round using a
  conforming wallet client.
- The vote chain operates as a public, verifiable ledger — anyone can run
  a monitoring node to audit.
- The system operates with partial validator availability.
- The capabilities that validators hold over the outcome of a round,
  including the ability to exclude transactions, are documented rather
  than left implicit.
- No organisation holds more than one of the three roles — validator,
  relay operator, key-share holder — any two of which together would
  allow it to select share reveals by option or to group and decrypt a
  voter's shares.


# Non-requirements

- Governance policy decisions such as proposal eligibility, quorum
  requirements, and fund disbursement rules (see ZIP 1016 [^zip-1016]).


# Specification

## System Overview

The coinholder voting system operates on a purpose-built Cosmos SDK vote
chain. Zcash mainnet snapshots provide the set of eligible Ironwood pool
balances.

The vote chain stores:

- A **Vote Commitment Tree** (VCT): a Poseidon Merkle tree of vote
  commitments.
- Three **nullifier sets**: governance nullifiers (alternate nullifiers
  from note claims), VAN nullifiers (from delegation consumption), and
  share nullifiers (from share reveals).
- An **encrypted share accumulator** per (proposal, option position):
  the running component-wise sum of the El Gamal ciphertexts at that
  position in every revealed share.
- The round's **final VCT root**, recorded when the round enters
  REVEALING, to which every share reveal in the round is anchored.

The vote chain verifies a zero-knowledge proof for each transaction type:
delegation, vote, and share reveal. The proof circuits are specified in
`draft-valargroup-shielded-voting` [^draft-voting-protocol].

## Deployment Architecture

A complete deployment consists of:

- **Vote chain nodes** — one or more `svoted` instances running CometBFT
  consensus.
- **Relays** — untrusted store-and-forward services that accept
  finished share reveal messages from clients that will not be online
  across their submission schedule, and submit each message at the time
  the client requested. These MUST be operated separately from the vote
  chain nodes and from the election authority key-share holders, and
  MUST NOT run in the `svoted` process; see [Relay Operator],
  [Role Separation] and [Why Roles Are Separated]. Earlier deployments
  bundled a submission server, which also constructed the reveal
  proofs, into the `svoted` binary. The rules a client follows in
  handing messages to relays are specified in the "Share Submission"
  section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol]; the relay's service interface is not yet
  specified in any normative document (see [Open Issues]).
- **Nullifier service** — a PIR server that provides private nullifier
  exclusion proofs to voters (see [Nullifier Service]).
- **Vote configuration document** — a per-round document published
  by the vote manager that lists the network endpoints of the vote
  chain nodes, relays and nullifier service operators participating in
  the round. The document format and distribution rules are specified
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

Validators participate in consensus and execute the chain's automatic
combination of partial decryptions into a tally (see the "Tally"
section of `draft-valargroup-shielded-voting` [^draft-voting-protocol]).
Each validator maintains two keypairs:

- **Consensus keypair**: used for CometBFT consensus.
- **Account keypair**: used for submitting chain transactions.

Validators join the network by following the flow described in
[Onboarding Validators].

Validators take no part in the election authority key ceremony and hold
no share of the election authority key; the ceremony is run by the
key-share holders (see [Key-Share Holder]). What validators can and
cannot do to a round through their control of transaction inclusion is
specified in the "Transaction Inclusion" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol].

A validator MUST NOT also be a relay operator or a key-share holder of
the same round; see [Role Separation].

### Relay Operator

A relay operator runs one or more relays (see
[Deployment Architecture]). A relay receives a finished share reveal
message together with the time at which the client asks for it to be
submitted, holds it until that time, and submits it to a vote chain
node unaltered. It receives no witness material and constructs no
proofs. A client hands a relay at most one share of any vote, over a
network connection used for no other share of that vote, as specified
in the "Share Submission" section of `draft-valargroup-shielded-voting`
[^draft-voting-protocol], which also states what a relay MUST and MUST
NOT require of the clients it serves.

A relay operator is trusted for availability only. It learns, from the
message it holds, what a chain observer learns from the same message
once it is recorded, plus the network origin of the client that handed
it over and the requested submission time. A relay holding one share of
a vote can group nothing. What a coalition of relay operators and
key-share holders could do with that metadata is why the role is
separated from key-share holding in [Role Separation].

The role is OPTIONAL: a client that is online across its submission
schedule submits its share reveal messages directly. Because a client
MUST select a distinct relay for each of its $N_s$ share reveal
messages, a deployment that offers relayed submission SHOULD make at
least $N_s$ relays available to clients; with fewer, a client can relay
only some of its shares and must submit the rest directly.

A relay operator MUST NOT also be a validator or a key-share holder of
the same round; see [Role Separation].

### Key-Share Holder

A key-share holder is an organisation admitted to a round's holder set.
The holders of a round run the election authority key ceremony among
themselves over the vote chain; each ends it holding a share of the
round's election authority private key, and no party ever holds the key
itself (see the "Election Authority Key Ceremony" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]).

Each key-share holder maintains two keypairs:

- **Account keypair**: used for submitting ceremony, complaint,
  acknowledgement and partial decryption transactions. Its address is
  the holder address the vote chain recognises for those transactions.
- **Pallas keypair**: used to receive, by ECIES, the shares dealt to
  the holder during the ceremony.

A key-share holder MUST NOT also be a validator or a relay operator of
the same round; see [Role Separation].

A key-share holder's duties for a round are to:

1. Take part in each round of the key ceremony within the published
   commitment, dealing, complaint and acknowledgement timeouts.
2. Verify the share it derives against its verification key, and
   acknowledge it on chain only if that check passes. The
   acknowledgement is the holder's ratification of the round (see the
   "Ratification" section of `draft-valargroup-shielded-voting`
   [^draft-voting-protocol]).
3. Erase the ceremony material the protocol requires it to erase once
   it has dealt its shares.
4. Submit a partial decryption, with its DLEQ proof, for each
   accumulator to be decrypted while the round is TALLYING.
5. Destroy its share at the end of the deployment's published retention
   period (see the "Election Authority Key Custody" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol]).

Key-share holders join by following the flow described in
[Onboarding Key-Share Holders].

A holder that causes a ceremony to fail by not participating is excluded
from the restarted ceremony by the protocol. A holder that repeatedly
causes ceremony restarts is removed from the holder set by the vote
manager. Each holder's participation record, and any removal from the
holder set, MUST be published.

### Nullifier Service Operator

A nullifier service operator runs a server from which wallet clients
MAY obtain Merkle non-membership proofs against the snapshot's
nullifier non-membership tree, for clients that do not construct those
proofs themselves (see [Participation Flow]). The role is OPTIONAL: a
deployment that expects its clients to construct their own proofs need
not include one. Where a deployment does include one, the retrieval
protocol it offers is a property of that deployment and is not specified
here.

### Role Separation

For a given round, the set of validators, the set of relay operators and
the set of key-share holders MUST be pairwise disjoint: no organisation
MAY hold more than one of the three roles in the same round. Which
capabilities each pair of roles would combine is set out in
[Why Roles Are Separated].

A deployment MUST publish which organisation operates each role for a
round (see [Deployment]), so that the independence of the three sets can
be checked rather than assumed.

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
   CometBFT consensus keypair and a Cosmos account keypair (see
   [Validator] for the role of each).

3. **Construct the genesis block.** Populate the genesis state
   with:
   - The chain identifier from step 1.
   - The vote manager singleton, set to an address controlled by
     the bootstrap operator (see [Bootstrap Operator]).
   - An initial balance for the vote manager account, in the
     chain's native token (`usvote`), sized to fund the planned
     validator set via subsequent authorized transfers.
   - The bootstrap operator's own validator entry, bonded with
     consensus voting power.
   - The election authority key ceremony timeouts (commitment,
     dealing, complaint and acknowledgement), which are chain
     parameters (see the "Election Authority Key Ceremony" section of
     `draft-valargroup-shielded-voting` [^draft-voting-protocol]).
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
   keypair and account keypair (see [Validator]).

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
   validator registration transaction, which wraps a standard
   Cosmos staking validator creation message.

The amount transferred at step 6 determines the new validator's
consensus voting power. Validators register no Pallas key: they take
no part in the EA key ceremony (see [Validator]). Key-share holders
register theirs by the flow in [Onboarding Key-Share Holders].

The reference implementation (see [Reference implementation])
includes an automated `join.sh` script that performs the above
steps. The script is parameterized by the vote configuration
document URL; the same script is used for any poll and is not
specialized per poll.

### Onboarding Key-Share Holders

The vote manager admits key-share holders. The holder set of a round is
fixed at round creation: the round creation transaction names the
admitted holders, and a party admitted afterwards receives no share for
that round and waits for the next (see the "Election Authority Key
Ceremony" section of `draft-valargroup-shielded-voting`
[^draft-voting-protocol]).

A new key-share holder joins by following these steps:

1. **Generate keypairs.** Generate the holder's account keypair and
   Pallas keypair (see [Key-Share Holder]).

2. **Apply for admission.** Send the vote manager the account address,
   the Pallas public key and the identity of the operating
   organisation, through a channel that authenticates the applicant.

3. **Wait for funding.** The vote manager reviews the application and
   funds the holder's account via an authorized transfer (see
   [Bootstrap Operator] for the funding mechanism), so that the holder
   can submit its ceremony and tally transactions. The amount confers
   no consensus voting power: a key-share holder is not a validator.

4. **Register on-chain.** Once funds are received, submit the
   key-share holder registration transaction, which binds the Pallas
   public key to the holder's address. From then on the vote chain
   recognises that address for the holder's ceremony, complaint,
   acknowledgement and partial decryption transactions.

5. **Admission to a round.** The vote manager names the registered
   holder in the holder set of a subsequently created round. The vote
   manager MUST NOT name a holder whose Pallas public key is not
   registered, since such a holder cannot receive the shares dealt to
   it and would cause the ceremony to fail.

The vote manager MUST publish the holder set of each round, with each
holder's registered Pallas public key, as part of the round's deployment
record (see [Deployment]); the protocol draft's "Deployment" section
lists the same items so that any party can recompute the ceremony's
verification keys. The vote manager MUST also publish each removal from
the holder set, with the participation record that motivated it (see
[Key-Share Holder]).

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
validator is producing blocks. New validators, relay operators and
nullifier service operators are added by proposing updates to the
document, which the vote manager reviews and applies; for validators
this proposal is automated by the join.sh flow described in
[Onboarding Validators], while relay operators and nullifier service
operators propose their entries directly.
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
   transaction, naming the round's key-share holder set (see
   [Onboarding Key-Share Holders]). Where a chain is bootstrapped for
   the round, the roots are also passed into genesis state (see
   [Genesis Validator Setup]).
3. **Attestation.** Each administrator reads both roots from a Zcash
   consensus node under its own control, confirms they match the
   round, and publishes its attestation. Reading the roots from the
   proposer, or comparing bytes against the proposer's document, is not
   attestation.
4. **Ceremony and ratification.** Key-share holders run the
   distributed key generation among themselves on the vote chain,
   within the published ceremony timeouts; each verifies its share and
   acknowledges it on chain. Those acknowledgements ratify the round,
   and it enters ACTIVE once at least the decryption threshold of them
   have been recorded. Validators take no part in the ceremony.
5. **Voting.** While the round is ACTIVE, until `vote_end_time`, the
   chain accepts delegation and vote transactions from coinholders
   (see [Coinholder Participation]).
6. **Reveal.** At `vote_end_time` the round enters REVEALING and the
   VCT is frozen at its final root. Until `reveal_end_time`, voters'
   clients construct their share reveal messages and submit them,
   directly or by handing them to relays; relay operators submit each
   message they hold at its requested time. A deployment SHOULD
   publish, from more than one operator, the count of share reveal
   transactions accepted into the mempool alongside the count included
   in blocks, as described in the "Transaction Inclusion" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol].
7. **Tally.** After `reveal_end_time`, key-share holders submit partial
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
Ironwood pool notes at the round's snapshot height (see
the "Snapshot Configuration" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]). Funds held in Sapling or transparent
pools at the snapshot height carry no voting weight; coinholders
who wish to participate with such funds MUST migrate them to
the Ironwood pool before the snapshot height.

### Wallet Setup

A wallet client obtains the vote configuration document for the
round (see [Vote Configuration Publication]), which lists the
vote chain endpoints, the relay endpoints, the nullifier service
endpoints, and the protocol versions in use. The configuration
document schema and the wallet-side validation rules are specified in
`draft-valargroup-shielded-voting-wallet-api`
[^draft-wallet-api].

### Participation Flow

For each Ironwood pool note the coinholder uses as voting weight, the
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

4. **Reveal the vote's shares.** At `vote_end_time` the round enters
   REVEALING and the VCT is frozen. The wallet must come back online at
   least once during the reveal window, before `reveal_end_time`: it
   obtains the round's final VCT root and a Merkle path for its Vote
   Commitment against that root, constructs a Vote Reveal Proof for
   each of the vote's $N_s$ shares, and draws a submission schedule.
   It then either submits the share reveal messages itself at the
   scheduled times, remaining online across the schedule, or hands
   each message, with its scheduled time, to a distinct relay (see
   [Relay Operator]) over a separate network connection per message.
   No witness material leaves the wallet in either case. A wallet that
   does not return during the reveal window loses its vote. Proof
   construction, the independence rules and the schedule are specified
   in the "Share Submission" and "Submission Timing" sections of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol].

After the round is finalized, the coinholder may verify the final
tally following [Verification and Auditing].

## Verification and Auditing

The vote chain is publicly readable. Any party running a full
node of the chain — a validator, the vote manager, or an
independent observer — can verify all aspects of a voting round
by replaying chain state and applying the verification
procedures defined in companion ZIPs. The checks a verifier MUST
perform, their order, and what each does and does not establish are
specified in the "Verification" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]; this
section states how a full-node operator carries them out.

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

- **Tally correctness.** A full-node operator confirms that every
  share reveal transaction in the round is anchored to the round's
  final VCT root, re-aggregates the position-$j$ ciphertexts of those
  transactions into a per-(`proposal_id`, option position $j$)
  accumulator for each option position, applies the DLEQ Proof
  Verification to each stored partial decryption, re-derives the
  Lagrange combination for each accumulator that is to be decrypted,
  and confirms the published per-option totals following the Tally
  procedure in `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].

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
the validator that decides which transactions enter blocks, the relay
operator that receives voters' finished share reveal messages, and the
holder of a share of the election authority key.

Earlier drafts bundled all three into one binary run by one operator
set, reasoning that validators already participated in the key
ceremony, so a separate submission server operator class would add a
trust assumption without clear benefit.

The reasoning treats a trust assumption as a cost to be minimised by
reducing the number of distinct parties. What matters is not how many
parties there are, but which capabilities land together. No single
role recovers a voter's balance or decision: a relay holding one share
of a vote can group nothing, a validator sees ciphertexts it cannot
open, and a key-share holder can, with enough peers, decrypt individual
shares but has no way to tell which belong to one vote. Each pair does
more than either alone:

- **Validators and key-share holders.** A share reveal exposes its
  proposal identifier but not its decision, so validators acting alone
  can exclude reveals only by proposal, arrival time, network origin,
  or wholesale. A coalition of validators and $t$ key-share holders
  could decrypt reveals as they arrive and exclude them by the option
  they support and by their weight.
- **Relay operators and key-share holders.** The protocol publishes
  nothing that groups the shares of one vote, so what a coalition
  could correlate is metadata: the network origin and requested
  submission time each relay sees. A coalition of relay operators
  pooling arrival logs with $t$ key-share holders could group a voter's
  shares by that metadata and decrypt them, recovering the balance the
  decomposition into shares exists to hide.

Held by the same organisation, each pair acts with no collusion
required. Separating the roles does not add an assumption. It restores
one that bundling had quietly removed, by making a coalition necessary
where previously a single operator sufficed. It also makes the
assumption checkable: with the operator of each role published, anyone
can verify that the three sets are disjoint, which is not possible when
one binary performs all three.

## Other Design Choices

**Separate vote chain (not Zcash mainnet)**: the vote chain is purpose-built
for governance with ZKP-optimized state transitions (Poseidon hashing, custom
transaction types). Zcash mainnet's transaction throughput and scripting model
are not designed for interactive multi-phase voting protocols.

**Ironwood-only snapshots**: the voting protocol is built on the
circuit-friendly primitives of the Orchard protocol (Poseidon hashing,
Pallas curve), which the Ironwood pool uses. Sapling and transparent
pools use incompatible cryptographic constructions.
The corresponding requirement on coinholders is stated in
[Eligibility].

**Cosmos SDK**: provides a mature BFT consensus engine (CometBFT),
validator lifecycle management (bonding, jailing for missed blocks,
consensus power distribution), and a
transaction pipeline that can be extended with custom message types and
ante handlers for ZKP verification. The alternative — building a chain
from scratch — would duplicate well-tested consensus infrastructure.

**Funding equals voting power**: bonding serves three purposes: it
determines which validators participate in consensus, it enables
jailing of inactive validators who miss blocks, and an even funding
split gives each validator a roughly equal probability
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
| The organisation operating each relay | Allows the role separation required in [Role Separation] to be checked. |
| The organisation holding each election authority key share, and its registered Pallas public key | As above; see [Onboarding Key-Share Holders]. |
| Each key-share holder's ceremony participation record, and any removal from the holder set | Makes persistent non-participation visible; see [Key-Share Holder]. |
| Stake distribution across validators | Determines the coalition size required to exclude transactions; see the "Transaction Inclusion" section of `draft-valargroup-shielded-voting` [^draft-voting-protocol]. |
| Software versions for the chain, circuits and client library | Required to reproduce or audit a round. |

The three operator sets MUST be disjoint (see [Role Separation]).
Round-level parameters — the decryption threshold and holder count, the
ceremony timeouts, the reveal window length, administrator keys and
threshold, confirmation depth, and the key-share retention period — are
published with the rules they parameterise, in the "Deployment" section
of `draft-valargroup-shielded-voting` [^draft-voting-protocol].


# Reference implementation

- [^ref-vote-sdk] — Cosmos SDK vote chain (`svoted`) implementing
  the chain-side state, ceremony, tally, and the submission server
  that the protocol replaces with relays.
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
  defaults — how it splits shares, which relays it uses, when it
  submits — are the protocol as every voter experiences it. The
  conditions under which a result may be described as representative are
  not specified. Requiring that some number of independent
  implementations be *available* is not checkable, and is met by two
  implementations one of which casts nearly every ballot. A checkable
  form would bound the share of ballots, or of voting weight, cast
  through any one implementation, and would bind the deployment that
  publishes the result.
- **Relay service interface**: the interface by which a client hands a
  share reveal message and its requested submission time to a relay
  (see [Relay Operator]) is not specified in any ZIP. The protocol
  draft places it with this document and constrains it — no client
  authentication, no identifier that persists across submissions, the
  message submitted unaltered — but does not define it. A normative
  form is needed before clients and relays from independent
  implementations can interoperate.


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
