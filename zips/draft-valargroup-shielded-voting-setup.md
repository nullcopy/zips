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

Bonded validator
: A validator whose stake is bonded under the vote chain's staking
  module [^cosmos-staking], so that it takes part in consensus and is
  eligible to produce blocks. See [Validator].

Ceremony key
: The public key a trustee registers on the vote chain, under which the
  encrypted shares dealt to it during the election authority key
  ceremony are addressed. See [Onboarding Trustees] and the "Election
  Authority Key Ceremony" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].

Decryption threshold ($t$)
: The number of trustees whose partial decryptions are needed to decrypt
  a voting round's tally. Fixed from the number of trustees, as
  specified in the "Election Authority Key Ceremony" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Election authority (EA)
: The keypair under which a voting round's vote shares are encrypted and
  whose private key decrypts the tally. A fresh keypair is generated for
  each voting round, in distributed form among its trustees, by the
  process in the "Election Authority Key Ceremony" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Election authority key ceremony
: The protocol, run on the vote chain among a voting round's trustees,
  that produces the round's election authority public key and each
  trustee's key share. See the "Election Authority Key Ceremony" section
  of `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Final VCT root
: The root of a voting round's Vote Commitment Tree at the moment the
  round leaves the ACTIVE state, to which every share reveal in the
  round is anchored. See the "Round Lifecycle" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Genesis state
: The state from which a vote chain starts, including its initial
  validator set and its chain parameters. See [Genesis].

Ironwood pool
: The Zcash shielded pool over which votes are weighted. The Ironwood
  pool uses the Orchard protocol: its notes, key hierarchy, note
  commitment tree, nullifiers, signatures and proving system are those
  of Orchard. References in this ZIP to Orchard keys, notes, nullifiers
  or circuits refer to those constructions as used in the Ironwood pool.

Nullifier service
: An OPTIONAL service from which a wallet MAY obtain a nullifier
  non-membership proof against a voting round's snapshot. See
  [Nullifier Service].

Nullifier service operator
: The entity that runs a nullifier service. See
  [Nullifier Service Operator].

Partial decryption
: A trustee's contribution to decrypting a voting round's tally,
  accompanied by a proof that it was computed with the trustee's key
  share. See the "Tally" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].

Poll runner
: The party that runs a voting round: it chooses the snapshot, reads
  the snapshot roots, names the round's trustees, creates the round on
  the vote chain, and publishes and signs the vote configuration. See
  [Poll Runner].

Poll signature
: The poll runner's signature over a voting round's defining fields, by
  which wallets recognise the round as the one the poll runner is
  running. See the "Poll Signature" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Ratification
: A trustee's published statement, made by acknowledging its key share
  on the vote chain, that it has verified the voting round's snapshot
  roots, holds a verified share, and will take part in the tally. See
  the "Ratification" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].

Relay
: An untrusted store-and-forward service to which a voter's wallet MAY
  hand a finished share reveal message, together with the time at which
  to submit it, for later submission to the vote chain. A relay
  constructs no proofs and receives no witness material. See
  [Relay Operator] and the "Share Submission" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Relay operator
: The entity that runs one or more relays. See [Relay Operator].

Reveal window
: The period, following the voting window, during which the vote chain
  accepts share reveal transactions for a voting round. See the "Round
  Lifecycle" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].

Share reveal message
: The finished message, carrying a Vote Reveal Proof, by which one
  encrypted share of a vote is revealed to the vote chain. See the
  "Share Reveal Message" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].

Snapshot
: The Zcash mainnet block, identified by height and hash, at which the
  eligible balances of the Ironwood pool are captured for a voting
  round. See the "Snapshot Configuration" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Snapshot height
: The Zcash mainnet block height of the snapshot.

Snapshot roots
: The Ironwood pool note commitment tree root and the nullifier
  non-membership tree root at the snapshot, carried in the round
  creation transaction. See the "Reading the Snapshot Roots" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Trustee
: One of the parties named in a voting round that jointly generate its
  election authority key and each hold a share of the private key.
  Trustees ratify the round and decrypt its tally. See [Trustee].

Validator
: An operator of the vote chain's consensus. See [Validator].

Vote Authority Note (VAN)
: A commitment on the Vote Commitment Tree that represents spendable
  voting authority, produced by delegation. See the "Data Structures"
  section of `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Vote chain
: The blockchain that records every voting round's delegation, vote,
  share reveal and ceremony transactions. See [What the Chain Records].

Vote chain node
: A node running the vote chain software, whether or not it is a
  validator. A vote chain node that is not a validator holds a full copy
  of the chain but takes no part in consensus.

Vote Commitment Tree (VCT)
: The append-only Merkle tree, maintained per voting round by the vote
  chain, whose leaves are Vote Authority Notes and Vote Commitments. See
  the "Data Structures" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].

Vote configuration
: The per-round document, published and signed by the poll runner, from
  which wallets learn a voting round's defining fields and the network
  endpoints through which to take part in it. Its format is specified in
  `draft-valargroup-shielded-voting-wallet-api` [^draft-wallet-api]; see
  [Vote Configuration Publication].

Voting round
: A bounded period during which a set of proposals is open for voting,
  anchored to a single Zcash mainnet snapshot and a fresh election
  authority key. See the "Voting Round" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Voting round identifier
: The unique identifier of a voting round, derived by the vote chain
  from the round creation transaction. See the "Poll Creation" section
  of `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Voting window
: The period during which the vote chain accepts delegation and vote
  transactions for a voting round. See the "Round Lifecycle" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

Zcash consensus node
: A node that validates the Zcash mainnet chain under the Zcash protocol
  and serves its state at a given block, including the snapshot roots.
  Any implementation that follows the Zcash protocol and serves both
  roots is acceptable.

For definitions of cryptographic terms including *alternate nullifier*,
*nullifier non-membership tree*, *nullifier domain*, *pool snapshot*, and
*claim*, see the Orchard Proof-of-Balance ZIP [^draft-balance-proof]. For
the remaining terms of the protocol — governance hotkey, governance
nullifier, share nullifier, vote commitment, vote share, and the terms of
the election authority key ceremony — see the "Terminology" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol].


# Abstract

This ZIP specifies how to set up and operate a vote chain for Zcash
shielded coinholder voting, and how to deploy and run a voting round on
it: the genesis state and validators of the chain; the parties that run
a voting round — the poll runner, the trustees, relay operators and
nullifier service operators — and how each is onboarded; how those
parties are kept separate so that no organisation both sees a voter's
share reveals arrive, or decides which of them enter blocks, and holds
key material that can decrypt them; and the procedures by which any
party audits a voting round.

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
  delegations, votes and share reveals, but not the network-layer
  metadata of their submission. A vote chain node sees the source
  address, submission time and connection of every transaction it
  receives, and a validator additionally sees the peer-to-peer
  propagation pattern of every transaction on the network.
- The vote chain is a public ledger. Transaction contents are
  encrypted or zero-knowledge-proven, but their existence, ordering and
  block-inclusion timing are a permanent public record.
- A relay operator learns, from each share reveal message handed to
  it, what a chain observer learns from the same message once it is
  recorded, plus the network origin of the wallet that handed it over
  and the requested submission time. See [Relay Operator] and the
  "Privacy Implications" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol].
- A nullifier service operator sees the source address and time of
  each query, which reveals that a wallet is preparing to take part in
  a voting round, even where the retrieval protocol conceals which
  nullifier was asked about. See [Nullifier Service].
- Validator onboarding reveals validator identities: a validator's
  network address is visible to its peers, and a deployment publishes
  the organisation operating each validator (see [Deployment]).
  Validators are not anonymous and are not designed to be.
- A trustee's retained key share is a live capability against the
  encrypted shares recorded permanently on the vote chain. The
  retention period a deployment publishes (see [Deployment]) is the
  period over which the protocol's amount-privacy claims hold; see the
  "Election Authority Key Custody" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].


# Requirements

- A deployment can set up and operate a vote chain by following
  [Operating the Vote Chain].
- A poll runner can deploy and conduct a voting round on a running vote
  chain by following [Running a Voting Round] and the referenced
  companion ZIPs.
- A Zcash coinholder with eligible Ironwood pool funds at a voting
  round's snapshot can take part in the voting round using a conforming
  wallet.
- The vote chain operates as a public, verifiable ledger: anyone can
  run a vote chain node to audit it.
- The vote chain operates with partial validator availability.
- No organisation acts as more than one of validator, relay operator
  and trustee for the same voting round.


# Non-requirements

- Governance policy decisions such as proposal eligibility, quorum
  requirements, and fund disbursement rules (see ZIP 1016 [^zip-1016]).
- The cryptographic constructions of the protocol, which are specified
  in `draft-valargroup-shielded-voting` [^draft-voting-protocol] and
  are not restated here.


# Specification

This specification has two parts. [Operating the Vote Chain] specifies
how a vote chain is set up and run: its genesis state, its validators
and what it records. [Running a Voting Round] specifies how a voting
round is deployed and conducted on such a chain: the parties that run
it, how they are onboarded, the steps of a voting round, how
coinholders take part, and how a voting round is verified. The chain
records voting rounds without knowledge of who is running them: any
number of voting rounds MAY be in progress on one chain at once, and
creating one requires no privileged role (see the "Poll Creation"
section of `draft-valargroup-shielded-voting` [^draft-voting-protocol]).

## Operating the Vote Chain

### What the Chain Records

The vote chain is a purpose-built blockchain that records, for each
voting round, the transactions the protocol defines — round creation,
trustee registration, ceremony, acknowledgement, delegation, vote,
share reveal and partial decryption — and maintains the per-round state
the protocol requires (see the "Vote Chain" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]):

- The **Vote Commitment Tree**: a Merkle tree of vote authority notes
  and vote commitments.
- Three **nullifier sets**: governance nullifiers (from delegation),
  VAN nullifiers (from voting) and share nullifiers (from share
  reveals).
- An **encrypted share accumulator** per (proposal, option position):
  the running sum of the encrypted shares revealed at that position.
- The round's **final VCT root**, recorded when the round enters
  REVEALING.
- The round's lifecycle state, its election authority key ceremony and
  the acknowledgements that ratify it.

The vote chain verifies a zero-knowledge proof for each delegation, vote
and share reveal transaction, as specified in
`draft-valargroup-shielded-voting` [^draft-voting-protocol]. All of
this state is kept per voting round, and the chain does not distinguish
voting rounds by who created them.

The reference implementation is a Cosmos SDK chain using CometBFT
consensus (see [Reference implementation]). Nothing in this document
depends on that choice beyond the validator lifecycle it provides.

### Genesis

A vote chain starts from a genesis state that provisions its initial
validator set. There is no operator role for genesis: whoever assembles
the genesis state and starts the initial validators' vote chain nodes
does so once, and holds no continuing authority over the chain.

The genesis state contains:

- The chain identifier, which distinguishes transactions signed for
  this vote chain from those of any other.
- The initial validators, each with its consensus public key and its
  bonded stake. The stake bonded to each validator determines its
  consensus voting power; an even distribution across validators
  reduces the risk of consensus capture.
- The election authority key ceremony timeouts (commitment, dealing,
  complaint and acknowledgement) and the maximum number of ceremony
  attempts, which are chain parameters (see the "Election Authority Key
  Ceremony" section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol]). A deployment MUST publish the values it
  applies (see [Deployment]).
- The standard module states the vote chain software requires (for the
  reference implementation, the Cosmos SDK auth, bank and staking
  modules).

The genesis state pre-populates no voting rounds, no trustee
registrations and no nullifier sets; all of these enter the chain
through transactions after it starts producing blocks. A deployment
MUST publish the genesis file and the network address of at least one
vote chain node, so that joining validators, and the vote chain nodes
that poll runners and wallets use, can find the chain.

### Validator

A validator runs a vote chain node that takes part in consensus,
producing and voting on blocks, and thereby determines which
transactions enter blocks. Validators execute every state transition
the chain defines, including the automatic combination of partial
decryptions into a tally (see the "Tally" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]).

Each validator maintains two keypairs:

- **Consensus keypair**: used to sign blocks and consensus votes.
- **Account keypair**: used to submit chain transactions, including
  its own registration.

Validators take no part in the election authority key ceremony and
hold no share of the election authority key. What validators can and
cannot do to a voting round through their control of transaction
inclusion is specified in the "Transaction Inclusion" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]; a
deployment MUST publish the stake distribution across validators, as
that section requires, and SHOULD publish the mempool and inclusion
counts it describes.

A validator also MUST NOT be a trustee or a relay operator of any
voting round recorded on the chain it validates; see
[Role Separation].

### Onboarding Validators

A validator joining the chain after genesis:

1. **Acquires the vote chain software**, from a published release of
   the reference implementation (see [Reference implementation]) or by
   building from source. A binary release MUST be verified against a
   published checksum before use.
2. **Obtains the genesis file and a peer**, from the addresses the
   deployment publishes (see [Genesis]), and initialises a local vote
   chain node directory.
3. **Generates its keypairs** (see [Validator]).
4. **Syncs the chain**: starts its vote chain node, connects to the
   published peers, and syncs to the current height.
5. **Bonds stake and registers on chain** by submitting the validator
   creation transaction with the stake bonded to its account. How a
   deployment provisions the stake of a joining validator is a matter
   for that deployment; the stake distribution that results MUST be
   published (see [Deployment]).

The stake bonded at step 5 determines the new validator's consensus
voting power. A validator registers no ceremony key: it takes no part
in the election authority key ceremony.

### Role Separation

For a given voting round, the set of validators, the set of relay
operators and the set of trustees MUST be pairwise disjoint: no
organisation MAY act in more than one of the three roles for the same
voting round. Since validators serve every voting round on the chain, a
validator also MUST NOT be a trustee or a relay operator of any voting
round on it. Why each pair is kept apart is set out in
[Why Roles Are Separated].

A deployment MUST publish which organisation operates each validator
and which organisation acts as each trustee of each voting round (see
[Deployment]), so that the disjointness of those two sets can be
checked rather than assumed. Relay operation is permissionless (see
[Relay Operator]), so a deployment cannot publish who operates every
relay; the rule binds validators and trustees, which MUST NOT operate
relays for the voting rounds they serve.

## Running a Voting Round

A voting round is created, run and tallied by parties that need no
privileged standing on the vote chain: the poll runner that creates and
signs it, the trustees that hold its key, the relay operators that
forward share reveal messages, and the nullifier service operators that
serve exclusion proofs. This part specifies each role, how it is
onboarded, and the steps of a voting round from snapshot to tally.

### Poll Runner

The poll runner is the party that runs a voting round. It chooses the
snapshot and reads the snapshot roots from a Zcash consensus node,
names the round's trustees, submits the round creation transaction
from an account it controls, and publishes and signs the vote
configuration by which wallets recognise the round (see the "Snapshot
Configuration", "Poll Creation" and "Poll Signature" sections of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]).

The poll runner holds no on-chain authority beyond that of the account
that created the round, and the chain does not distinguish it: any
account MAY create a voting round, and a round no trustee ratifies
never opens. What makes a party the poll runner of a round, from a
wallet's point of view, is that the vote configuration carries a valid
poll signature under a key the wallet recognises. A deployment MUST
publish the poll runner of each voting round and its signing key (see
[Deployment]).

The poll runner is responsible for the coordination a round needs
outside the chain: ensuring that any nullifier service the round lists
has ingested the snapshot before the round opens (see
[Nullifier Service]), and that the trustees it names are registered
and prepared to serve within the ceremony timeouts (see
[Onboarding Trustees]). The poll runner MAY also be one of the round's
trustees.

### Trustee

A trustee is one of the $n$ independent parties named in a voting
round's creation transaction. The trustees of a round run the election
authority key ceremony among themselves over the vote chain; each ends
it holding a share of the round's election authority private key, and
no party ever holds the key itself (see the "Election Authority Key
Ceremony" section of `draft-valargroup-shielded-voting`
[^draft-voting-protocol]).

A trustee is not the poll runner under another name. The poll runner
is the single party that configures a voting round and signs its vote
configuration; the trustees are the $n$ parties, independent of one
another, that jointly generate and hold its decryption key, and every
one of them must ratify the round before it opens. The poll runner MAY
be one of the trustees it names. A trustee also MUST NOT be a validator
or a relay operator of the same voting round; see [Role Separation].

Each trustee maintains two keypairs:

- **Account keypair**: used to submit ceremony, complaint,
  acknowledgement and partial decryption transactions. Its address is
  the trustee address the vote chain recognises for those
  transactions.
- **Ceremony key**: registered on the vote chain, under which the
  encrypted shares dealt to the trustee during the ceremony are
  addressed (see [Onboarding Trustees]).

A trustee's duties for a voting round are to:

1. Take part in each stage of the election authority key ceremony
   within the published commitment, dealing, complaint and
   acknowledgement timeouts, and in each restarted attempt.
2. Read the round's snapshot roots from a Zcash consensus node under
   its own control and confirm that they match the round, as specified
   in the "Reading the Snapshot Roots" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol]. Reading
   the roots from the poll runner, or comparing them with the vote
   configuration, does not satisfy this.
3. Verify the key share it derives as the ceremony specifies, and
   acknowledge it on chain only if that check and the check in step 2
   both pass. The acknowledgement is the trustee's ratification of the
   round (see the "Ratification" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol]); the
   round opens only once every trustee has ratified it.
4. Erase the ceremony material the protocol requires it to erase once
   it has dealt its shares.
5. Submit a partial decryption, with its proof, for each accumulator to
   be decrypted while the round is TALLYING.
6. Destroy its key share at the end of the deployment's published
   retention period (see the "Election Authority Key Custody" section
   of `draft-valargroup-shielded-voting` [^draft-voting-protocol]).

A trustee that does not want to serve a round it is named in simply
does not ratify it, and the round never opens. Every trustee's
participation in every stage and attempt of a ceremony is a matter of
chain record; a poll runner is under no obligation to name again a
trustee that has caused ceremonies to fail.

### Onboarding Trustees

Trustee registration is permissionless. Any account MAY submit a
trustee registration transaction, which binds a ceremony key to the
account's address; from then on the vote chain recognises that address
for the ceremony, complaint, acknowledgement and partial decryption
transactions of any voting round that names it. A party becomes a
trustee of a voting round by being named in that round's creation
transaction (see [Poll Runner]); the trustee set of a round is fixed at
creation, and a party registered afterwards receives no share for that
round.

A prospective trustee:

1. **Generates its keypairs**: an account keypair and a ceremony key
   (see [Trustee]).
2. **Funds its account** so that it can submit its ceremony and tally
   transactions. Provisioning trustee accounts is an operational matter
   for whoever runs the poll, and confers no consensus voting power: a
   trustee is not a validator.
3. **Registers on chain** by submitting the trustee registration
   transaction.
4. **Gives the poll runner** its account address and the identity of
   the operating organisation, through a channel that authenticates
   it, so that the poll runner can name it and publish it.

A poll runner MUST NOT name a trustee whose ceremony key is not
registered: the chain rejects the round creation transaction (see the
"Poll Creation" section of `draft-valargroup-shielded-voting`
[^draft-voting-protocol]), and such a trustee could not receive the
shares dealt to it.

A deployment MUST publish the trustee set of each voting round, with
each trustee's operating organisation, account key and ceremony key,
as part of the round's deployment record (see [Deployment]), so that
any party can check role separation, verify acknowledgements, and
recompute the ceremony's verification keys.

### Relay Operator

A relay operator runs one or more relays. A relay receives a finished
share reveal message together with the time at which the wallet asks
for it to be submitted, holds it until that time, and submits it to a
vote chain node unaltered. It receives no witness material and
constructs no proofs. A wallet hands a relay at most one share of any
vote, over a network connection used for no other share of that vote,
as specified in the "Share Submission" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol], which also
states what a relay MUST and MUST NOT require of the wallets it serves.

Relay operation is permissionless: anyone MAY run a relay, and a wallet
MAY use any relay it is configured with. A poll runner MAY list relays
it knows of in the vote configuration (see
[Vote Configuration Publication]); a poll runner that does so SHOULD
list at least $N_s$ of them, operated independently of one another,
because a wallet MUST select a distinct relay for each of its $N_s$
share reveal messages and with fewer can relay only some of its shares.
Listing a relay is not a statement about who operates it, and a
wallet's choice is not limited to the list.

A relay operator is trusted for availability only. It learns, from the
message it holds, what a chain observer learns from the same message
once it is recorded, plus the network origin of the wallet that handed
it over and the requested submission time. A relay holding one share of
a vote can group nothing.

The role is OPTIONAL: a wallet that is online across its submission
schedule submits its share reveal messages directly. A relay operator
also MUST NOT be a validator or a trustee of a voting round for which
it relays; see [Role Separation].

### Nullifier Service Operator

A nullifier service operator runs a server from which wallets MAY
obtain Merkle non-membership proofs against a voting round's nullifier
non-membership tree, for wallets that do not construct those proofs
themselves (see [Participation Flow]). The role is OPTIONAL: a
deployment that expects its wallets to construct their own proofs need
not include one. Where a deployment does include one, the retrieval
protocol it offers is a property of that deployment and is not
specified here.

### Nullifier Service

The nullifier service is an OPTIONAL service, external to the vote
chain, from which a wallet MAY obtain a proof that a note's nullifier
is absent from the Ironwood pool nullifier set at a voting round's
snapshot, for wallets that do not construct such proofs themselves
(see [Participation Flow]). It serves proofs by private information
retrieval, so that a query reveals neither which nullifier is being
checked nor the answer to any observer of the network.

Where a deployment runs one, it is run by a nullifier service operator
(see [Nullifier Service Operator]) and operates as a three-stage
pipeline:

1. **Ingest**: fetch the Ironwood pool nullifier set up to the snapshot
   height from a Zcash consensus node and persist it to local storage.
   The ingest pipeline MUST follow the chain whose block at the snapshot
   height has the snapshot's block hash, tracking chain reorganisations
   so that the tree built in the next stage is that of the snapshot.
2. **Export**: build the nullifier non-membership tree as specified in
   `draft-valargroup-orchard-balance-proof` [^draft-balance-proof] and
   export whatever query structures its retrieval protocol requires, so
   that the server can restart without rebuilding the tree from raw
   nullifiers. The root of the tree the service builds MUST equal the
   round's nullifier non-membership tree root. The service is not the
   source of that root — the poll runner reads it from a Zcash
   consensus node, as specified in the "Reading the Snapshot Roots"
   section of `draft-valargroup-shielded-voting`
   [^draft-voting-protocol] — and a service whose root differs serves
   proofs the round will not accept.
3. **Serve**: accept queries from wallets and return responses. The
   operator gives the poll runner the service's address for inclusion
   in the vote configuration (see [Vote Configuration Publication]).

The query and response wire format is not yet specified in any
normative document. See [Open Issues].

### Vote Configuration Publication

The poll runner publishes a vote configuration for each voting round it
runs, in the format and through the distribution channel specified in
`draft-valargroup-shielded-voting-wallet-api` [^draft-wallet-api], and
signs it with its poll signature (see the "Poll Signature" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]). The
configuration carries the round's defining fields and the network
endpoints a wallet needs: vote chain nodes to query and submit
through, nullifier services, and any relays the poll runner lists.
Wallets fetch the configuration from that channel; the vote chain
itself provides no service discovery.

The poll runner MAY publish and sign the configuration as soon as the
round is created, since the signature does not cover the election
authority key. A wallet does not take the configuration on trust: it
verifies the snapshot roots against a Zcash consensus node of its own
and the election authority key against the trustees' acknowledgements
on the vote chain, as the "Poll Signature" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol] specifies.

How wallet implementers learn a poll runner's signing key and
distribution channel is out-of-band: the poll runner announces both
before the round opens, and a deployment publishes the signing key
(see [Deployment]). One vote chain MAY carry the voting rounds of
several poll runners at once, each with its own configuration.

### Conducting a Voting Round

The rules a voting round follows — its proposals and snapshot, its
creation, the poll signature, its lifecycle states, the election
authority key ceremony and the unanimous ratification that opens
voting — are consensus rules of the vote chain and rules for wallets,
specified in the "Voting Round" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]. This
section states who carries out each step.

1. **Snapshot.** The poll runner chooses the snapshot block and reads
   both snapshot roots from a Zcash consensus node, as specified in the
   "Reading the Snapshot Roots" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol]. It does
   not derive or recompute them; any Zcash consensus node that follows
   the Zcash protocol and serves the roots is acceptable. It
   coordinates with each nullifier service operator the round will
   list so that their ingest and export pipelines (see
   [Nullifier Service]) have run to the snapshot before the round
   opens.
2. **Creation.** The poll runner submits the round creation
   transaction, carrying the snapshot, the roots, the proposals, the
   deadlines and the names of the trustees (see [Onboarding Trustees]
   and the "Poll Creation" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol]). The
   round enters PENDING and the ceremony starts automatically.
3. **Configuration.** The poll runner publishes and signs the vote
   configuration (see [Vote Configuration Publication]). Wallets verify
   the poll signature, verify the roots against a Zcash consensus node
   of their own, and, once the ceremony completes, verify the election
   authority key against the trustees' acknowledgements (see the "Poll
   Signature" section of `draft-valargroup-shielded-voting`
   [^draft-voting-protocol]).
4. **Ceremony and ratification.** The trustees run the election
   authority key ceremony among themselves on the vote chain within
   the published timeouts. Each verifies the snapshot roots against a
   Zcash consensus node under its own control, verifies its key share,
   and acknowledges it on chain. The acknowledgements ratify the round;
   it enters ACTIVE only once every trustee has ratified it. A round
   whose ceremony fails on its final attempt, or that some trustee
   never ratifies, never opens. Validators take no part in the
   ceremony.
5. **Voting.** While the round is ACTIVE, until `vote_end_time`, the
   chain accepts delegation and vote transactions from coinholders
   (see [Coinholder Participation]).
6. **Reveal.** At `vote_end_time` the round enters REVEALING and the
   VCT is frozen at its final root. Until `reveal_end_time`, wallets
   construct their share reveal messages and submit them, directly or
   through relays; relay operators submit each message they hold at its
   requested time. A deployment SHOULD publish, from more than one
   operator, the mempool and inclusion counts described in the
   "Transaction Inclusion" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol], which
   also states what validators can and cannot do to a round at this
   stage.
7. **Tally.** After `reveal_end_time` the round enters TALLYING; the
   trustees submit partial decryptions, the chain combines them and
   publishes the tally, and the round is FINALIZED. A round in which
   fewer than $t$ partial decryptions arrive within the chain's tally
   timeout finalizes without a tally.

### Coinholder Participation

For a Zcash coinholder to participate in a voting round, they must
satisfy an eligibility precondition and follow a wallet-driven
participation flow. The protocol details for each step are specified in
companion ZIPs; this section ties them together from the coinholder's
perspective.

#### Eligibility

Voting weight derives from the value held in a coinholder's Ironwood
pool notes at the voting round's snapshot height (see the "Snapshot
Configuration" section of `draft-valargroup-shielded-voting`
[^draft-voting-protocol]). Funds held in Sapling or transparent pools
at the snapshot height carry no voting weight; coinholders who wish to
participate with such funds MUST migrate them to the Ironwood pool
before the snapshot height.

#### Wallet Setup

A wallet obtains the vote configuration for the voting round (see
[Vote Configuration Publication]), which lists the vote chain nodes,
the nullifier services, any relays the poll runner lists, and the
protocol versions in use. The configuration format, the wallet-side
validation rules and the verification of the poll signature are
specified in `draft-valargroup-shielded-voting-wallet-api`
[^draft-wallet-api]. Before taking part, the wallet verifies the
round's snapshot roots against the Zcash consensus node it trusts for
Zcash state, and the election authority key against the trustees'
acknowledgements, as specified in the "Poll Signature" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol].

#### Participation Flow

For each Ironwood pool note the coinholder uses as voting weight, the
wallet performs:

1. **Obtain a non-membership proof.** Prove that the note's standard
   Orchard nullifier is absent from the set of nullifiers revealed at
   or before the snapshot height, against the snapshot's nullifier
   non-membership tree root. A wallet holding that nullifier set
   constructs the proof itself, as specified in
   `draft-valargroup-orchard-balance-proof` [^draft-balance-proof].
   A wallet that does not hold the set MAY obtain the proof from a
   nullifier service (see [Nullifier Service]); doing so reveals to
   that service which nullifier was asked about, and therefore which
   note, unless its retrieval protocol conceals the query, so a wallet
   SHOULD construct its own proof where it can.

2. **Submit a delegation transaction.** Construct a Delegation Proof
   asserting ownership of the eligible note without revealing which
   note it is, and submit the transaction to a vote chain node listed
   in the vote configuration. This produces a Vote Authority Note (VAN)
   on the Vote Commitment Tree. The proof construction and the on-chain
   handling are specified in the "Delegation Phase" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol].

For each proposal the coinholder votes on, the wallet performs:

3. **Submit a vote transaction.** Construct a Vote Proof consuming the
   current VAN and producing a new VAN with the relevant proposal
   authority bit cleared, plus a Vote Commitment binding the chosen
   option, as specified in the "Vote Phase" section of
   `draft-valargroup-shielded-voting` [^draft-voting-protocol].

4. **Reveal the vote's shares.** At `vote_end_time` the voting round
   enters REVEALING and the VCT is frozen. The wallet must come back online at
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

After the voting round is finalized, the coinholder may verify the
tally following [Verification and Auditing].

### Verification and Auditing

The vote chain is publicly readable. Any party running a vote chain
node — a validator, a poll runner, a trustee or an independent observer
— can verify a voting round by replaying chain state. The checks a
verifier MUST perform, their order, and what each does and does not
establish are specified in the "Verification" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol]; this
section states how the operator of a vote chain node carries them out,
in three layers.

- **Per-transaction proof verification.** Each delegation, vote and
  share reveal transaction carries a zero-knowledge proof that the
  chain checks at inclusion against the out-of-circuit verification
  rules of the "Delegation Proof", "Vote Proof" and "Vote Reveal Proof"
  sections of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol]. Those rules apply the claim proof
  verification and the non-membership checks of
  `draft-valargroup-orchard-balance-proof` [^draft-balance-proof]
  transitively, so replaying every transaction re-verifies all three
  layers of proof at once.

- **Nullifier set integrity.** The chain maintains three nullifier
  sets — governance nullifiers (from delegation), VAN nullifiers (from
  voting) and share nullifiers (from share reveals) — and rejects any
  transaction that would re-use an already published nullifier. The
  verifier confirms set integrity by replaying every accepted
  transaction and checking that no nullifier appears twice.

- **Tally correctness.** The verifier confirms that every share reveal
  transaction in the round is anchored to the round's final VCT root,
  re-aggregates the revealed ciphertexts into a per-(proposal, option
  position) accumulator for each option position, verifies the proof
  of each stored partial decryption, and recomputes the published
  per-option totals from the partial decryptions following the tally
  procedure in the "Tally" section of
  `draft-valargroup-shielded-voting` [^draft-voting-protocol].

The three layers verify that the transactions in a round are well
formed and correctly accumulated *with respect to* the round's snapshot
roots. They do not establish that those roots are correct: the roots
are supplied as input at round creation and are not validated by
consensus. Confirming them requires reading both roots from a Zcash
consensus node under the verifier's own control and comparing them
with the round's, by the procedure in the "Reading the Snapshot Roots"
section of `draft-valargroup-shielded-voting` [^draft-voting-protocol],
which also establishes that the round is well formed. A verification of
a round is incomplete without it.

A round that finalized without a tally (see the "Round Lifecycle"
section of `draft-valargroup-shielded-voting` [^draft-voting-protocol])
verifies as having no tally; this is itself a verifiable property of
chain state.

The procedures above verify the transactions that are present. They
cannot establish that no transaction is missing; see the "Transaction
Inclusion" section of `draft-valargroup-shielded-voting`
[^draft-voting-protocol].


# Rationale

## Why Roles Are Separated

Three roles are held by different organisations: the validator that
decides which transactions enter blocks, the relay operator that
receives voters' finished share reveal messages, and the trustee that
holds a share of the election authority key. No single role recovers a
voter's balance or decision. Two pairs do:

- **Validators and $t$ trustees** could decrypt share reveals as they
  arrive and exclude them by the option they support and by their
  weight. Validators alone can exclude only by proposal, by arrival
  time, by network origin or wholesale (see the "Transaction Inclusion"
  section of `draft-valargroup-shielded-voting`
  [^draft-voting-protocol]).
- **Relay operators and $t$ trustees** could group a voter's shares by
  the metadata relays see — network origin and requested submission
  time — and decrypt them, recovering the balance that the
  decomposition into shares exists to hide.

Held by one organisation, either pair acts with no collusion required.
Keeping the roles apart makes a coalition necessary, and publishing the
validators and trustees of a voting round makes their disjointness
checkable.

## Other Design Choices

**Separate vote chain**: the vote chain is purpose-built for
governance, with state transitions and transaction types designed
around the protocol's proofs. Zcash mainnet's transaction throughput
and scripting model are not designed for an interactive multi-phase
voting protocol.

**Ironwood-only snapshots**: the protocol's proofs are built on the
constructions of the Orchard protocol, which the Ironwood pool uses;
Sapling and transparent funds cannot be proven under them. The
corresponding requirement on coinholders is stated in [Eligibility].

**A general-purpose consensus framework**: the reference vote chain is
built on the Cosmos SDK, which provides a BFT consensus engine,
validator lifecycle management (bonding, jailing for missed blocks,
consensus power distribution) and a transaction pipeline that can be
extended with the protocol's transaction types and proof verification.

**Stake determines voting power**: bonding determines which validators
take part in consensus, enables jailing of validators that miss blocks,
and, where stake is evenly split, gives each validator a roughly equal
chance of proposing a block — not important for correctness, but
important for liveness.

**Permissionless voting rounds and registrations**: the chain grants no
account the right to create voting rounds or to admit trustees, so
there is no privileged key whose compromise would let an attacker deny
or misconfigure a voting round for everyone, and several poll runners
can use one chain at once. What makes a voting round genuine to a
wallet is the poll signature and the trustees' acknowledgements, both
of which the wallet verifies, not the account that created it.


# Deployment

A deployment MUST publish the following. These are recorded here,
alongside the properties that depend on them, so that a divergence
between a published value and the specification is visible to a reader
of this document.

| Parameter | Why it is published |
|---|---|
| The organisation operating each validator, and the stake distribution across validators | Establishes the validator set and the coalition size required to exclude transactions; see the "Transaction Inclusion" section of `draft-valargroup-shielded-voting` [^draft-voting-protocol]. |
| The genesis file and the network address of at least one vote chain node | Lets validators, poll runners and wallets find the chain; see [Genesis]. |
| The election authority key ceremony timeouts and the maximum number of ceremony attempts | Chain parameters fixed in the genesis state; see [Genesis] and the "Election Authority Key Ceremony" section of `draft-valargroup-shielded-voting` [^draft-voting-protocol]. |
| For each voting round, the organisation acting as each trustee, its account key and its ceremony key | Allows the separation required in [Role Separation] to be checked, lets wallets verify acknowledgements, and lets any party recompute the ceremony's verification keys; see [Onboarding Trustees]. |
| For each voting round, the poll runner and its signing key | Establishes whose poll signature wallets recognise; see [Vote Configuration Publication]. |
| For each voting round, the reveal window length | Bounds the period in which a wallet must return to reveal; see the "Round Lifecycle" section of `draft-valargroup-shielded-voting` [^draft-voting-protocol]. |
| The retention period for trustees' key shares | Bounds the period over which amount-privacy claims hold; see the "Election Authority Key Custody" section of `draft-valargroup-shielded-voting` [^draft-voting-protocol]. |
| Software versions for the chain, circuits and client library | Required to reproduce or audit a voting round. |

The validators and the trustees of a voting round MUST be disjoint, and
validators and trustees MUST NOT operate relays for it (see
[Role Separation]). The "Deployment" section of
`draft-valargroup-shielded-voting` [^draft-voting-protocol] lists the
protocol parameters, and the round parameters above alongside the
rules they parameterise; a deployment publishes each value once, and
the two lists are to be read as one.


# Reference implementation

- [^ref-vote-sdk] — Cosmos SDK vote chain implementing the chain-side
  state, the election authority key ceremony and the tally.
- [^ref-nullifier-pir] — PIR server and client for privately
  retrieving nullifier non-membership proofs.


# Open Issues

- **Stake provisioning for joining validators**: how a validator that
  joins after genesis obtains stake to bond (see
  [Onboarding Validators]) is left to the deployment. A normative
  mechanism, or a requirement on what the deployment publishes about
  it beyond the resulting stake distribution, is not specified.
- **PIR client-server wire format**: the query and response wire
  format used between wallets and the nullifier service
  (see [Nullifier Service]) is not currently specified in any
  ZIP. The PIR draft scopes its outer transport out of remit, and
  the wallet API ZIP describes only the endpoint URLs. A normative
  spec home is needed before wallets and nullifier service
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
- **Relay service interface**: the interface by which a wallet hands a
  share reveal message and its requested submission time to a relay
  (see [Relay Operator]) is not specified in any ZIP. The protocol
  draft places it with this document and constrains it — no client
  authentication, no identifier that persists across submissions, the
  message submitted unaltered — but does not define it. A normative
  form is needed before wallets and relays from independent
  implementations can interoperate.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-1016]: [ZIP 1016: Community and Coinholder Funding Model](zip-1016.md)

[^draft-balance-proof]: [Draft ZIP: Orchard Proof-of-Balance](draft-valargroup-orchard-balance-proof.md)

[^draft-voting-protocol]: [Draft ZIP: Zcash Shielded Voting Protocol](draft-valargroup-shielded-voting.md)

[^draft-wallet-api]: [Draft ZIP: Shielded Voting Wallet API](draft-valargroup-shielded-voting-wallet-api.md)



[^cosmos-staking]: [Cosmos SDK `x/staking` module documentation](https://docs.cosmos.network/main/build/modules/staking)

[^ref-vote-sdk]: [valargroup/vote-sdk: Cosmos SDK vote chain for shielded voting](https://github.com/valargroup/vote-sdk)

[^ref-nullifier-pir]: [valargroup/vote-nullifier-pir: PIR system for nullifier non-membership proofs](https://github.com/valargroup/vote-nullifier-pir)
