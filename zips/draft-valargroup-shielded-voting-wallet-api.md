```
ZIP: Unassigned
Title: Shielded Voting Wallet API
Owners: Dev Ojha <dojha@berkeley.edu>
        Adam Tucker <adamleetucker@outlook.com>
        Roman Akhtariev <ackhtariev@gmail.com>
        Greg Nagy <greg@dhamma.works>
Status: Draft
Category: Standards / Wallet
Created: 2026-03-24
License: MIT
Pull-Request: <https://github.com/zcash/zips/pull/1244>
```

# Terminology

The key words "MUST", "MUST NOT", "SHOULD", and "MAY" in this document
are to be interpreted as described in BCP 14 [^BCP14] when, and only
when, they appear in all capitals.

The terms below are to be interpreted as follows:

Acknowledgement

: A trustee's vote chain transaction, signed by its trustee account
  key, committing to a round's election authority public key. A wallet
  retrieves acknowledgements via [Round Acknowledgements] and accepts
  an `ea_pk` only when every trustee has acknowledged it. See
  [Binding to the Chain Round].

Delegation

: The act of proving ownership of unspent Orchard notes in the Ironwood
  pool at the snapshot height and registering a vote authority note on
  the vote commitment tree. See [^orchard-balance-proof].

Express reveal

: A voter's per-vote choice to have the vote's $N_s$ share reveal
  messages submitted within a single session rather than across the
  reveal window, at a stated privacy cost. See [Submission Timing] and
  the "Share Submission" section of [^voting-protocol].

Final VCT root

: The root of a round's vote commitment tree after the last effective
  delegation or vote transaction recorded at or before the round's
  `vote_end_height`. Every share reveal message in the round is
  anchored to it. See the "Round Lifecycle" section of
  [^voting-protocol].

Ironwood pool

: The Zcash shielded pool over which votes are weighted. The Ironwood
  pool uses the Orchard protocol; references in this document to
  Orchard keys, notes, nullifiers, signatures or circuits refer to
  those constructions as used in the Ironwood pool. See
  [^voting-protocol].

Nullifier exclusion proof

: A Merkle non-membership proof, against the snapshot's nullifier
  non-membership tree, that a note was unspent at the snapshot. A
  wallet constructs one locally or retrieves one as specified in
  [Nullifier Exclusion Proof Retrieval].

Poll runner

: The party that runs a voting round and signs the vote configuration
  by which wallets find it. A wallet recognises a poll runner by a
  public key held in the wallet's own configuration. See
  [Configuration Authentication] and the "Poll Signature" section of
  [^voting-protocol].

Poll signature

: The poll runner's signature over a round's defining fields, carried
  in the configuration's `poll_signature` field and verified as
  specified in [Configuration Authentication].

Proposals hash

: The hash of a round's proposals, computed as specified in
  [Proposals Hash], by which the configuration's proposals are bound
  to the chain round and to the poll signature.

Reveal material

: The private values a wallet must retain from vote construction until
  every share of that vote has been revealed, listed in
  [Reveal Material Persistence].

Reveal window

: The range of vote chain heights, following the voting window, within
  which a round's share reveal transactions are effective. It ends at
  the round's `reveal_end_height`. See the "Round Lifecycle" section of
  [^voting-protocol].

Share

: One of the $N_s$ encrypted fragments of the holder's ballot count
  within a vote commitment. Each share is revealed independently
  during the reveal window by a share reveal message that the wallet
  constructs itself.

Share nullifier

: The value a share reveal publishes to prevent a share being counted
  twice, derived as specified in [Share Nullifier].

Share reveal message

: The message a share reveal transaction carries: a Vote Reveal Proof,
  a share nullifier, the option-vector ciphertexts, the proposal
  identifier, the final VCT root and the round identifier. Specified in
  the "Share Reveal Transaction" section of [^voting-protocol]; its
  JSON encoding is given in [Share Reveal Message Format].

Trustee

: One of the parties named in a round's configuration that jointly
  generate the round's election authority key and each hold a share of
  its private key. Every trustee ratifies the round by acknowledging
  that key on the vote chain; a wallet verifies those acknowledgements
  before encrypting to it. See [Binding to the Chain Round] and the
  "Election Authority Key Ceremony" section of [^voting-protocol].

Trustee account key

: The Ed25519 key under which a trustee signs its vote chain
  transactions, including its acknowledgement. Its public key is the
  `account_pk` of the trustee's configuration entry.

Vote authority note (VAN)

: A note appended to the vote commitment tree during delegation. The VAN
  carries the delegated vote weight and is consumed (nullified) when
  the holder casts a vote.

Vote commitment

: A commitment, appended to the vote commitment tree when a vote is
  cast, that binds the vote's encrypted shares, proposal and decision.
  Its opening is part of the reveal material.

Vote commitment tree

: An append-only Merkle tree, derived per round from the effective
  delegation and vote transactions on the vote chain, that holds vote
  authority notes and vote commitments. The tree root at a given block
  height serves as a public input to zero-knowledge proof verification.

Vote configuration

: The JSON document, signed by the poll runner, by which a wallet
  discovers a vote round and the services that serve it. See
  [Vote Configuration Format].

Vote round

: A voting session, bounded by vote chain heights, defining a set of
  proposals, a Zcash snapshot, a voting deadline and a reveal deadline.
  Wallet clients interact with exactly one vote round at a time. The
  round states and their transitions are specified in the "Round
  Lifecycle" section of [^voting-protocol].

Vote server

: A vote chain node exposing the chain query and transaction
  submission endpoints specified in this document. What a vote server
  reports about a round is its own derivation from the vote chain's
  record, which it serves as a convenience; vote servers are not
  authenticated. See [Binding to the Chain Round].

Zcash consensus node

: A node that validates the Zcash chain under the Zcash protocol. The
  wallet's Zcash consensus node, or the light client backend it trusts
  for Zcash state, is the source against which it verifies a round's
  snapshot roots. See [Snapshot Verification].

# Abstract

This ZIP specifies the REST API endpoints, wire formats, and discovery
mechanism that wallet clients use to participate in shielded on-chain
voting rounds. It covers vote round discovery via a per-vote
configuration document signed by the round's poll runner, the checks by
which a wallet verifies a round's snapshot roots and election authority
key for itself, data query endpoints for reading chain state,
transaction submission endpoints for delegation, vote casting and share
reveal, the client-side rules on which the protocol's privacy claims
depend, and the encoding conventions for all exchanged data.

# Motivation

The shielded voting protocol involves multiple ZIPs that specify the
cryptographic circuits, share submission and election authority key
ceremony [^voting-protocol], proof-of-balance [^orchard-balance-proof],
and the operational setup of a deployment [^voting-setup]. A wallet
integrator currently must read several of these
specifications to understand which endpoints to call, what wire formats
to use, and how to discover an active vote.

This ZIP consolidates the wallet-facing API surface into a single
document, specifying the REST endpoints, JSON wire formats, encoding
conventions, and discovery mechanism needed to participate in a vote.

Versioning fields in the vote configuration allow the protocol to evolve
(new PIR schemes, circuit versions, tally methods) while maintaining
compatibility with wallets already released.

# Requirements

- A wallet can discover and join an active voting round using a published
configuration document.
- A wallet can submit delegation and vote commitment transactions
using the wire formats in this specification. Proof construction is
specified in companion ZIPs.
- A wallet can submit its own share reveal transactions, at heights it
draws and over network paths of its own, without disclosing to any
third party which shares belong to the same vote or which option the
vote supports.
- A wallet retains, from the moment it casts a vote until every share
of that vote is revealed, the material the Vote Reveal Proofs need,
and that material never leaves the device.
- A voter can choose, per vote and after being told the cost, to have
that vote revealed in a single session; the wallet never makes that
choice for the voter.
- A wallet can authenticate a configuration document, not merely check
that it is well formed.
- A wallet verifies a round's snapshot roots against the Zcash
consensus node it already trusts, and the round's election authority
key against the trustees' own acknowledgements, rather than accepting
either on any third party's word.
- A voter can delegate part of their balance rather than all of it.
- Network-level requirements needed for the protocol's privacy claims
to hold at the client are stated normatively, not left to
implementers.
- Each protocol component (vote server, vote protocol, tally method,
PIR) can be versioned and upgraded independently. A change to one
component has no impact on other components or the configuration schema.

# Non-requirements

- Trustee onboarding and the EA key ceremony (distributed key
generation), specified in [^voting-protocol] and [^voting-setup].
- The vote chain's consensus and block production, and the rules by
which any party interprets its record, which are specified in
[^voting-protocol]. A vote server applies those rules to serve the
endpoints below; a wallet MAY apply them itself instead.
- Round creation and the poll runner's operations.

# High level summary

This section is non-normative.

This section provides an informational overview of the end-to-end
sequence a wallet follows to participate in a shielded voting round.
Each step references the normative section that specifies its details.
All requirements use the language defined in those sections.

The vote configuration itself is versioned by `config_version`, which
tracks the schema of the configuration document. It also carries four
independently versioned protocol components:

- **`vote_protocol`** — the ZKP circuits (ZKP1, ZKP2, ZKP3) and
  commitment tree structure. The circuits are designed to be
  upgradeable: a new circuit version bumps `vote_protocol` without
  affecting the other components.
- **`tally`** — threshold decryption and result aggregation.
- **`pir`** — the nullifier PIR retrieval scheme.
- **`vote_server`** — the vote server REST API through which a wallet
  reads chain state and submits transactions.

See [Version Handling] for the normative rules.

## Discovery and Validation

1. **Obtain vote configuration.** Fetch or receive the vote
   configuration JSON document for the round.
   See [Vote Configuration Format].

2. **Validate configuration.** Check all fields against the rules in
   [Validation Rules] and verify version compatibility per
   [Version Handling]. Reject the configuration and stop if any check
   fails.

3. **Verify the poll signature.** Verify `poll_signature` against a
   recognised poll runner key per [Configuration Authentication].
   Reject the configuration and stop if it does not verify.

4. **Verify the snapshot roots.** Confirm `snapshot_blockhash`,
   `nc_root` and `nullifier_imt_root` against the wallet's own Zcash
   consensus node per [Snapshot Verification]. Stop if any differs.

5. **Fetch active round from chain.** Query
   `GET /shielded-vote/v1/rounds/active` to confirm the round is ACTIVE
   (or, for a wallet returning to reveal, REVEALING) and retrieve
   on-chain parameters, including `ea_pk`. A round opens once every
   trustee has acknowledged its election authority key. See
   [Active Round].

6. **Bind the round to the configuration.** Confirm that the chain's
   round carries exactly the values the poll runner signed, including
   the proposals hash computed from the configuration's `proposals`
   array, and that every trustee in the configuration has acknowledged
   the round's `ea_pk` (fetched via [Round Acknowledgements]). See
   [Binding to the Chain Round].

## Delegation

7. **Obtain nullifier exclusion proofs.** Obtain Merkle
   non-membership proofs for the wallet's Orchard note nullifiers at
   `snapshot_height`. A wallet holding the nullifier set at that
   height constructs them itself; otherwise it retrieves them from a
   `pir_endpoints` server. See
   [Nullifier Exclusion Proof Retrieval].

8. **Construct and submit delegation transaction.** Build the ZKP1
   proof (proving Orchard note ownership at the snapshot height) and
   submit via `POST /shielded-vote/v1/delegate-vote`.
   See [Delegation Transaction] and [^orchard-balance-proof].

9. **Poll for delegation confirmation.** Poll
   `GET /shielded-vote/v1/tx/{hash}` using the `tx_hash` from the
   submission response until the response includes a non-empty `height`
   and `code` = 0. See [Confirmation Polling].

10. **Sync commitment tree and locate VAN.** Query the
    [Commitment Tree Leaves] endpoint to incrementally sync the local
    tree. Identify the wallet's vote authority note by its commitment
    `van_cmx` computed during step 8.

## Voting (during ACTIVE; repeat for each proposal)

11. **Construct and submit vote commitment.** Build the ZKP2 proof
    (consuming the current VAN and producing a new VAN) and submit
    via `POST /shielded-vote/v1/cast-vote`. The tree root at the
    anchor height is a public input to this proof. The vote decision
    is a private input to this proof and never appears in any request
    body. See [Vote Commitment Transaction] and [^voting-protocol].

12. **Persist the reveal material.** Store, encrypted at rest, the
    vote commitment and everything the Vote Reveal Proofs will need
    to open it during the reveal window. See
    [Reveal Material Persistence].

13. **Poll for vote commitment confirmation.** Poll
    `GET /shielded-vote/v1/tx/{hash}` until confirmed.

14. **Sync commitment tree.** Query [Commitment Tree Leaves] again
    to locate the new VAN (needed as input for the next proposal)
    and the vote commitment leaf, recording its leaf position.

Steps 11 through 14 are repeated sequentially for each proposal in
the round. Each iteration consumes the current VAN and produces a
new one, so proposals cannot be voted on in parallel. Voting ends
at `vote_end_height`.

## Share Reveal (during REVEALING)

The reveal window opens after `vote_end_height` and closes at
`reveal_end_height`. Share reveal messages cannot be constructed
before it opens, because they are anchored to the round's final VCT
root, which exists only once voting has closed. A wallet that is not
opened during the reveal window cannot reveal its shares, and the vote
is not counted. [Submission Timing] requires wallets to surface this
to the user.

15. **Obtain the final VCT root and Merkle path.** Query
    [Commitment Tree (Latest)] once the round is REVEALING, sync any
    remaining leaves via [Commitment Tree Leaves], and compute the
    Merkle path for each of the wallet's vote commitments against the
    final root.

16. **Construct the share reveal messages.** For each vote, construct
    $N_s$ Vote Reveal Proofs locally from the persisted reveal
    material and assemble $N_s$ share reveal messages. No party other
    than the wallet constructs these proofs. See [Share Submission]
    and [^voting-protocol].

17. **Draw the submission schedule.** Draw $N_s$ submission heights
    within the reveal window as specified in [Submission Timing], and
    persist the finished messages with their heights.

18. **Submit.** Submit each message via
    `POST /shielded-vote/v1/reveal-share` when the chain reaches its
    drawn height, from a background session where the platform allows,
    each over its own network path. On each later open during the
    window, reconcile and catch up as [Submission Timing] specifies.
    A voter who has chosen express reveal for the vote has all $N_s$
    messages submitted within this session instead. See
    [Share Submission] and [Network Isolation].

19. **Optionally confirm inclusion.** A wallet MAY check
    `GET /shielded-vote/v1/share-status/{roundId}/{nullifier}`, subject
    to the privacy caveats in [Share Status]. This is not a required
    step.

## Results (optional)

20. **View tally results.** Once the round has a tally, query
    `GET /shielded-vote/v1/tally-results/{round_id}` for the vote
    server's decrypted per-proposal tallies. See [Tally Results].

# Specification

## Vote Discovery

A vote configuration is a JSON document published for each vote round.
It contains all parameters a wallet needs to locate services and
participate in the round.

### Vote Configuration Format

```json
{
  "config_version": 4,
  "vote_round_id": "<hex, 64 characters>",
  "vote_servers": [
    {"url": "https://vote1.example.com", "label": "node-1"}
  ],
  "pir_endpoints": [
    {"url": "https://pir1.example.com", "label": "pir-1"}
  ],
  "snapshot_height": 2800000,
  "snapshot_blockhash": "<base64, 32 bytes>",
  "nc_root": "<base64, 32 bytes>",
  "nullifier_imt_root": "<base64, 32 bytes>",
  "vote_end_height": 1200000,
  "reveal_end_height": 1300000,
  "block_time_seconds": 6,
  "proposals": [
    {
      "id": 1,
      "title": "Approve protocol upgrade",
      "description": "Approve or oppose the proposed protocol upgrade.",
      "options": [
        {"index": 0, "label": "Support"},
        {"index": 1, "label": "Oppose"}
      ]
    }
  ],
  "trustees": [
    {"label": "trustee-1", "account_pk": "<base64, 32 bytes>"},
    {"label": "trustee-2", "account_pk": "<base64, 32 bytes>"}
  ],
  "supported_versions": {
    "pir": ["v0", "v1"],
    "vote_protocol": "v0",
    "tally": "v0",
    "vote_server": "v1"
  },
  "poll_signature": {
    "key_id": "poll-runner-1",
    "alg": "ed25519",
    "sig": "<base64, 64 bytes>"
  }
}
```

### Field Definitions


| Field                              | Type             | Description                                                                                                                    |
| ---------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `config_version`                   | integer          | Schema version of this configuration document. This specification defines version 4.                                           |
| `vote_round_id`                    | string           | Hex-encoded 32-byte vote round identifier (64 characters, lowercase).                                                          |
| `vote_servers`                     | array            | One or more vote server base URLs serving the chain query and transaction submission endpoints. Each entry has `url` (string) and `label` (string). |
| `pir_endpoints`                    | array            | One or more nullifier PIR server base URLs. Each entry has `url` and `label`.                                                  |
| `snapshot_height`                  | integer          | Zcash block height at which the Ironwood pool snapshot was taken.                                                              |
| `snapshot_blockhash`               | string           | Base64-encoded 32-byte hash of the Zcash block at `snapshot_height`.                                                           |
| `nc_root`                          | string           | Base64-encoded 32-byte Ironwood pool note commitment tree root at the snapshot.                                                |
| `nullifier_imt_root`               | string           | Base64-encoded 32-byte nullifier non-membership tree root at the snapshot.                                                     |
| `vote_end_height`                  | integer          | Vote chain block height after which delegation and vote transactions are no longer effective; the round is REVEALING above it. |
| `reveal_end_height`                | integer          | Vote chain block height after which share reveal transactions are no longer effective; the round is TALLYING above it.        |
| `block_time_seconds`               | number           | The vote chain's target block interval in seconds: the chain parameter the deployment published for the chain, as specified in the "Genesis" section of [^voting-setup]. Informational and not signed. A wallet uses it to present heights as estimated times and to size its submission schedule; see [Submission Timing]. |
| `proposals`                        | array            | Ordered list of proposals. Each has `id` (integer, 1-indexed), `title` (string), `description` (string), and `options` (array of `{index, label}`). |
| `trustees`                         | array            | The round's trustees, in the order the round creation transaction names them and the order hashed by [Trustees Hash]. Each entry has `label` (string) and `account_pk` (base64, the 32-byte Ed25519 trustee account key under which the trustee signs its chain transactions). |
| `supported_versions.pir`           | array of strings | PIR retrieval scheme versions supported by the servers (e.g., `["v0", "v1"]`).                                                 |
| `supported_versions.vote_protocol` | string           | Vote protocol version covering the ZKP circuits and commitment tree structure (e.g., `"v0"`).                                  |
| `supported_versions.tally`         | string           | Tally method version covering threshold decryption and result aggregation (e.g., `"v0"`).                                      |
| `supported_versions.vote_server`   | string           | Vote server version covering the REST API (e.g., `"v1"`).                                                                      |
| `poll_signature`                   | object           | The poll signature. Has `key_id` (string), `alg` (string) and `sig` (base64). See [Configuration Authentication].              |


### Validation Rules

The rules in this section are structural only: they establish that a
configuration document is well formed, not that it is authentic. A
document passing every check below may have been produced by anyone.

A wallet MUST additionally authenticate the configuration as specified
in [Configuration Authentication], and verify its snapshot roots as
specified in [Snapshot Verification]. A wallet MUST NOT use a
configuration that fails either, and MUST NOT treat the structural
checks below as a substitute for them.

A wallet MUST validate the structure of the configuration before use:

- `config_version` MUST be a version the wallet recognizes. This
specification defines version 4 and no other; a wallet MUST NOT accept
a document of any other version under this specification.
- `vote_round_id` MUST be exactly 64 lowercase hexadecimal characters.
- `vote_servers` MUST contain at least one entry.
- `pir_endpoints` MUST contain at least one entry.
- `snapshot_height` MUST be greater than 0.
- `reveal_end_height` MUST be greater than `vote_end_height`. The
minimum separation the protocol requires is specified in the "Round
Lifecycle" section of [^voting-protocol].
- `block_time_seconds` MUST be a positive number.
- `snapshot_blockhash`, `nc_root` and `nullifier_imt_root` MUST each be
the base64 encoding of exactly 32 bytes.
- `trustees` MUST contain at least 2 entries. Each entry MUST have a
string `label` and an `account_pk` that is the base64 encoding of
exactly 32 bytes. `account_pk` values MUST be unique across entries.
- `poll_signature` MUST be an object with a string `key_id`, a string
`alg`, and a base64 `sig`.
- `proposals` MUST contain between 1 and 50 entries.
- Each proposal MUST have between 2 and 8 options.
- Proposal `id` values MUST be unique and in the range 1 to 50.
- Option `index` values within a proposal MUST be unique and 0-indexed.
- The wallet MUST check version compatibility as specified in
[Version Handling]. In summary: `supported_versions.vote_server`,
`supported_versions.vote_protocol`, and `supported_versions.tally`
MUST be recognized versions; `supported_versions.pir` MUST contain
at least one version the wallet supports.

### Configuration Authentication

A wallet holds a **recognised poll runner set**: a list of poll runner
public keys, each with a `key_id` and an `alg`. The set is part of the
wallet's own configuration, not of any vote configuration document; how
it is provisioned is outside the scope of this specification. A key
absent from the set is not a poll runner as far as the wallet is
concerned, whatever a configuration says about it.

A configuration carries exactly one signature, `poll_signature`. To
verify it, a wallet:

1. MUST resolve `poll_signature.key_id` to a key in its recognised poll
   runner set. If no key matches, the signature is invalid.
2. MUST verify that `poll_signature.alg` matches the `alg` of the
   resolved key. If they differ, the signature is invalid.
3. MUST verify `poll_signature.sig` over the bytes defined in the
   "Poll Signature" section of `draft-valargroup-shielded-voting`
   [^voting-protocol]: the ASCII domain separator
   `ZcashVotingPollSignature:v4` (27 bytes), then `vote_round_id`
   (32 bytes, decoded from hex), `snapshot_height` (8 bytes,
   big-endian unsigned), `snapshot_blockhash`, `nc_root` and
   `nullifier_imt_root` (32 bytes each, decoded from base64),
   `proposals_hash` (32 bytes, computed from the configuration's
   `proposals` per [Proposals Hash]), `vote_end_height` and
   `reveal_end_height` (8 bytes each, big-endian unsigned), and
   `trustees_hash` (32 bytes, computed from the configuration's
   `trustees` per [Trustees Hash]). The version in the domain
   separator is the document's `config_version`. This specification
   defines one algorithm, `"ed25519"`, verified per RFC 8032
   [^rfc8032].

A wallet MUST accept a configuration only if all three steps succeed.

The signature establishes that the round — its snapshot, its
proposals, its deadlines and its trustees — is the one the poll runner
is running. It does not establish that the snapshot roots are correct
or that the election authority key is genuine, and a wallet MUST NOT
treat it as doing so: the wallet checks the roots itself
([Snapshot Verification]) and the key against the trustees'
acknowledgements ([Binding to the Chain Round]). One signature suffices
because nothing more is claimed by it. The signed bytes do not include
`ea_pk`, which does not exist when the poll runner signs, nor the
endpoint lists or `block_time_seconds`, which are conveniences; a
configuration can therefore be signed and verified before the round
opens.

### Trustees Hash

`trustees_hash` is the 32-byte BLAKE2b-256 hash, with personalization
`ZcashVoteTrustee` (16 bytes), of the concatenation of the `account_pk` values
of the configuration's `trustees` entries, each decoded from base64 to
its 32 bytes, in array order and without length prefixes, as specified
in the "Trustees Hash" section of [^voting-protocol]. For $n$ trustees
the input is exactly $32n$ bytes. `label` does not enter the hash: the
poll signature binds the trustees by their keys, and a wallet
identifies a trustee's acknowledgement by its `account_pk`, not by its
label.

### Snapshot Verification

The poll signature establishes which round the poll runner is running,
not that the round's snapshot roots are correct, and no third party
vouches for the roots on the wallet's behalf. Before taking part in a
round — before delegating, voting or submitting any share — a wallet
MUST perform the procedure in the "Reading the Snapshot Roots" section
of [^voting-protocol] against the Zcash consensus node it trusts for
Zcash state, which is the node or light client backend it syncs from:

1. Confirm that the block at `snapshot_height` on that node's best
   chain has hash `snapshot_blockhash`.
2. Obtain from that node the Ironwood pool note commitment tree root
   as of the end of that block — the pool's anchor at that height —
   and compare it with the configuration's `nc_root`.
3. Obtain from that node the root of the nullifier non-membership tree
   over every Ironwood pool nullifier revealed at or before that block,
   constructed as specified in [^orchard-balance-proof] and served by
   the node's API, and compare it with the configuration's
   `nullifier_imt_root`.

If the block hash differs or either root differs, the wallet MUST NOT
take part in the round. Agreement between the configuration and the
chain round ([Binding to the Chain Round]), or with any other party's
copy of the round, does not satisfy this requirement: it shows only
that two parties received the same values, not that the values are
correct.

This step requires the wallet's Zcash consensus node or light client
backend to serve the nullifier non-membership tree root at a given
height. The interface by which it does so is a Zcash-node-side API
outside this document, and is not yet specified anywhere; see
[Open issues]. A wallet whose backend does not serve it cannot complete
this step and therefore cannot take part in the round.

### Binding to the Chain Round

A wallet learns a round's on-chain parameters from a vote server
([Active Round]), and vote servers are not authenticated. An
authenticated configuration therefore constrains the round a wallet
takes part in only if the wallet checks that the two agree; and the
election authority key, which the configuration does not carry, is
authenticated only if the wallet checks it against the trustees.

**Equality.** Before delegating, voting, or submitting any share in a
round, a wallet MUST confirm that the `VoteRound` it retrieved carries
the same `vote_round_id`, `snapshot_height`, `snapshot_blockhash`,
`nc_root`, `nullifier_imt_root`, `vote_end_height` and
`reveal_end_height` as the authenticated configuration, that its
`proposals_hash` equals the hash of the configuration's `proposals`
computed per [Proposals Hash], and that its `trustees` carry the
configuration's `account_pk` values in the same order. A wallet MUST
NOT take part in a round that fails this check.

**Election authority key.** `ea_pk` is derived from the ceremony's
recorded commitments when the key ceremony completes and is not part
of the configuration; the wallet reads it from the `VoteRound`, which
is the vote server's derivation. Before taking part in a round, and in
any case before encrypting anything to `ea_pk`, a wallet MUST fetch
the round's acknowledgement transactions via [Round Acknowledgements]
and verify that for every entry of the configuration's `trustees`, at
index $i$ (1-based), there is an acknowledgement such that:

1. its `payload` parses as an acknowledgement transaction per the
   "Acknowledgement Transaction" section of [^voting-protocol], with
   `voting_round_id` equal to the round's, `trustee` equal to $i$, and
   `ea_pk` equal to the round's `ea_pk`;
2. its `signature` is a valid Ed25519 signature [^rfc8032] by that
   trustee's `account_pk` over the ASCII string `ZcashVoteTx:v1`
   followed by every byte of `payload` preceding the signature, as
   specified in the "Signed Payloads" section of [^voting-protocol].

A wallet MUST NOT encrypt to an `ea_pk` for which this check fails for
any trustee, and MUST NOT take part in the round. An `ea_pk`
acknowledged by every trustee under its own account key is the key
those trustees hold shares of; see the "Poll Signature" and
"Ratification" sections of [^voting-protocol]. A wallet MAY go further
and derive `ea_pk` itself from the round's recorded ceremony
commitments, in which case the acknowledgement check confirms that the
trustees hold shares of the key it derived.

Without the equality check a vote server can supply snapshot roots or
deadlines other than those the poll runner signed. Without the
acknowledgement check it could supply an `ea_pk` of its own, to which
the wallet would then encrypt every share; it cannot produce
acknowledgements of that key without the trustees' account keys.

### Distribution

The vote configuration is published out-of-band for each vote round.
Distribution mechanisms include:

- A developer-merged pull request to a well-known repository linking
to the configuration file.
- A CDN or API endpoint serving the configuration.
- Bundling the configuration within a wallet release.

The choice of distribution mechanism is outside the scope of this
specification. Regardless of the mechanism, the wallet MUST validate and
authenticate the configuration, and verify its snapshot roots, as
described above before using it.

## Data Query Endpoints

All query endpoints are served relative to a `vote_servers` base URL
from the vote configuration. Responses are JSON-serialized protobuf
messages; byte fields are base64-encoded.

### Active Round

```
GET /shielded-vote/v1/rounds/active
```

Returns the active voting round, if any. The `VoteRound` structure is
the vote server's derivation from the record, by the rules in
[^voting-protocol]; it carries the round creation transaction's fields
and the state the server derives from the transactions that followed.

**Response body:** A JSON object containing a `round` field with the
`VoteRound` structure:


| Field                | Type              | Description                                                          |
| -------------------- | ----------------- | -------------------------------------------------------------------- |
| `vote_round_id`      | base64 (32 bytes) | Round identifier.                                                    |
| `snapshot_height`    | uint64            | Zcash snapshot block height.                                         |
| `snapshot_blockhash` | base64 (32 bytes) | Zcash block hash at snapshot.                                        |
| `proposals_hash`     | base64 (32 bytes) | Hash of the proposals (see [Proposals Hash]).                        |
| `vote_end_height`    | uint64            | Vote chain height after which the round is REVEALING.               |
| `reveal_end_height`  | uint64            | Vote chain height after which the round is TALLYING.                |
| `nullifier_imt_root` | base64 (32 bytes) | Nullifier non-membership tree root.                                  |
| `nc_root`            | base64 (32 bytes) | Ironwood pool note commitment tree root.                             |
| `status`             | string            | One of `PENDING`, `ACTIVE`, `REVEALING`, `TALLYING`, `CANCELLED`, `FAILED`, as the server derives it at its current height; see the "Round Lifecycle" section of [^voting-protocol]. |
| `active_height`      | uint64            | Height at which the round became ACTIVE; 0 while PENDING.            |
| `ea_pk`              | base64 (32 bytes) | Election authority public key (compressed Pallas point); empty until the ceremony completes. |
| `proposals`          | array             | Proposals with `id` (uint32), `title`, `description`, and `options`. |
| `description`        | string            | Human-readable round description.                                    |
| `title`              | string            | Short human-readable round title.                                    |
| `creator_pk`         | base64 (32 bytes) | The Ed25519 key that signed the round creation transaction.          |
| `created_at_height`  | uint64            | Vote chain block height at which the round creation transaction was recorded. |
| `stage_window`       | uint32            | Blocks per ceremony stage.                                           |
| `max_attempts`       | uint32            | Ceremony attempts allowed.                                           |
| `trustees`           | array             | Trustees named in the round creation transaction, in order; each has `account_pk` and `ceremony_pk` (base64, 32 bytes each). A wallet checks this field against the configuration per [Binding to the Chain Round]. |


The response may contain additional fields related to the EA key
ceremony and threshold decryption (e.g., ceremony progress, trustee
commitments, encrypted dealings). These fields exist for trustee
coordination during distributed key generation and have no bearing on
wallet operations, so they are not documented here. The trustees'
acknowledgements, which a wallet does check, are served separately by
[Round Acknowledgements]. See the "Election Authority Key Ceremony"
section of [^voting-protocol] and
`draft-valargroup-shielded-voting-setup` [^voting-setup] for details.

The `proposals` field in the VoteRound response contains the same
proposals as the vote configuration document. The `proposals_hash`
field can be used to verify consistency between the two sources.

### Proposals Hash

`proposals_hash` is the BLAKE2b-256 hash, with personalization
`ZcashVoteProposl`, of the byte encoding of the round's proposals as
they appear in the round creation transaction, specified in the
"Proposals Hash" and "Round Creation Transaction" sections of
[^voting-protocol]. To compute it from a configuration's `proposals`
array, a wallet re-encodes the array by those rules: a `compactSize`
count, then for each proposal its `id` (one byte), `title` and
`description` (each a `compactSize` length and UTF-8 bytes), and its
options as a `compactSize` count followed by each option's `index`
(one byte) and `label` (a `compactSize` length and ASCII bytes), in
array order.

Wallets MUST verify `proposals_hash` against the vote configuration
before proceeding; see [Binding to the Chain Round].

If no active round exists, the response contains a `round` field with
a null or empty value.

### Round Details

```
GET /shielded-vote/v1/round/{round_id}
```

Returns details for a specific vote round.

**Path parameters:**

- `round_id`: Hex-encoded 32-byte vote round identifier (64 characters).

**Response body:** Same `VoteRound` structure as [Active Round].

### Round Acknowledgements

```
GET /shielded-vote/v1/rounds/{roundId}/acknowledgements
```

Returns the acknowledgement transactions recorded for a round: the
transactions by which its trustees ratified it at the end of the key
ceremony (see the "Election Authority Key Ceremony" and "Ratification"
sections of [^voting-protocol]).

**Path parameters:**

- `roundId`: Hex-encoded 32-byte vote round identifier (64 characters).

**Response body:** A JSON object containing an `acknowledgements`
array. Each entry represents one recorded acknowledgement transaction:

| Field       | Type              | Description                                                                 |
| ----------- | ----------------- | --------------------------------------------------------------------------- |
| `trustee`   | uint32            | The acknowledging trustee's index, 1-based, as the payload names it.        |
| `height`    | uint64            | Vote chain height at which the transaction was recorded.                    |
| `payload`   | base64 (variable) | The complete acknowledgement transaction payload, as specified in the "Acknowledgement Transaction" section of [^voting-protocol]. |
| `signature` | base64 (64 bytes) | The payload's Ed25519 signature, also present at the end of `payload`.      |

The array is empty for a round no trustee has yet acknowledged. It
MAY contain acknowledgements the protocol treats as ineffective, such
as those of a failed attempt. The server does not verify entries on the
wallet's behalf: the wallet MUST apply the checks in
[Binding to the Chain Round] to each entry and MUST NOT infer anything
from an entry's presence alone.

### List Rounds

```
GET /shielded-vote/v1/rounds
```

Returns all stored vote rounds.

**Response body:** A JSON object containing a `rounds` array of
`VoteRound` structures.

### Commitment Tree (Latest)

```
GET /shielded-vote/v1/commitment-tree/{round_id}/latest
```

Returns the latest commitment tree state for a round.

**Path parameters:**

- `round_id`: Hex-encoded 32-byte vote round identifier.

**Response body:** A JSON object containing a `tree` field:


| Field                | Type              | Description                                   |
| -------------------- | ----------------- | --------------------------------------------- |
| `next_index`         | uint64            | Next leaf index to be written.                |
| `root`               | base64 (32 bytes) | Current Merkle root.                          |
| `height`             | uint64            | Block height at which this root was computed. |
| `next_index_at_root` | uint64            | `next_index` at the time the root was stored. |


### Commitment Tree at Height

```
GET /shielded-vote/v1/commitment-tree/{round_id}/{height}
```

Returns the commitment tree state at a specific block height.

**Path parameters:**

- `round_id`: Hex-encoded 32-byte vote round identifier.
- `height`: Block height (decimal integer).

**Response body:** Same `CommitmentTreeState` structure as
[Commitment Tree (Latest)].

### Commitment Tree Leaves

```
GET /shielded-vote/v1/commitment-tree/{round_id}/leaves?from_height=X&to_height=Y
```

Returns commitment tree leaves appended in blocks within the specified
height range. Used by wallet clients to incrementally sync the local
copy of the vote commitment tree.

**Path parameters:**

- `round_id`: Hex-encoded 32-byte vote round identifier.

**Query parameters:**

- `from_height`: Start block height (inclusive).
- `to_height`: End block height (inclusive).

**Response body:** A JSON object containing a `blocks` array. Each
entry represents one block:


| Field         | Type                       | Description                                                         |
| ------------- | -------------------------- | ------------------------------------------------------------------- |
| `height`      | uint64                     | Block height.                                                       |
| `start_index` | uint64                     | Index of the first leaf appended in this block.                     |
| `leaves`      | array of base64 (32 bytes) | Commitment leaves (Pallas base field elements, little-endian each). |


### Tally Results

```
GET /shielded-vote/v1/tally-results/{round_id}
```

Returns the vote server's decrypted tally for a vote round, computed
from the recorded partial decryptions as specified in the "Tally"
section of [^voting-protocol]. Available once the server has observed
effective partial decryptions from at least $t$ trustees. The result
is the server's computation; any party can recompute it from the
record, and a wallet that presents it SHOULD say whose it is.

**Path parameters:**

- `round_id`: Hex-encoded 32-byte vote round identifier.

**Response body:** A JSON object containing a `results` array:


| Field           | Type              | Description                          |
| --------------- | ----------------- | ------------------------------------ |
| `vote_round_id` | base64 (32 bytes) | Round identifier.                    |
| `proposal_id`   | uint32            | Proposal identifier.                 |
| `vote_decision` | uint32            | Option position whose aggregate this entry reports. It is a property of the aggregate, not of any voter. |
| `total_value`   | uint64            | Decrypted aggregate value, in ballots (see the "Tally units" section of [^voting-protocol]). |


### Transaction Status

```
GET /shielded-vote/v1/tx/{hash}
```

Returns the confirmation status of a previously submitted transaction.

**Path parameters:**

- `hash`: Hex-encoded transaction hash.

**Response body:**


| Field    | Type   | Description                                                            |
| -------- | ------ | ---------------------------------------------------------------------- |
| `height` | string | Block height at which the transaction was recorded (empty if pending). |
| `code`   | uint32 | Result code (0 = recorded).                                            |
| `log`    | string | Error message if `code` is non-zero.                                   |
| `events` | array  | ABCI events emitted by the transaction.                                |

A recorded transaction is not necessarily effective: the chain records
without applying the protocol's rules (see the "Vote Chain Record"
section of [^voting-protocol]). A vote server MAY report, in `events`
or an additional field, whether it derives the transaction as
effective; a wallet that relies on that is relying on the server's
derivation.


## Delegation Transaction

The delegation transaction registers a holder's vote weight on the vote
commitment tree. It corresponds to ZKP1 (the delegation circuit) as
specified in [^orchard-balance-proof] and [^voting-protocol].

### Delegation Endpoint

```
POST /shielded-vote/v1/delegate-vote
```

### Delegation Request Body

A JSON object with the following fields:


| Field                   | Type                            | Description                                                                |
| ----------------------- | ------------------------------- | -------------------------------------------------------------------------- |
| `rk`                    | base64 (32 bytes)               | Randomized spend authorization verification key (compressed Pallas point). |
| `spend_auth_sig`        | base64 (64 bytes)               | RedPallas spend authorization signature.                                   |
| `signed_note_nullifier` | base64 (32 bytes)               | Nullifier of the dummy signed note.                                        |
| `cmx_new`               | base64 (32 bytes)               | Output note commitment (extracted x-coordinate).                           |
| `van_cmx`               | base64 (32 bytes)               | Vote authority note commitment (extracted x-coordinate).                   |
| `gov_nullifiers`        | array of base64 (32 bytes each) | Governance nullifiers, exactly 5, in note slot order.                      |
| `rt_cm`                 | base64 (32 bytes)               | Note commitment tree root, equal to the round's `nc_root`.                 |
| `rt_excl`               | base64 (32 bytes)               | Non-membership tree root, equal to the round's `nullifier_imt_root`.       |
| `proof`                 | base64 (variable)               | Halo 2 ZKP1 proof.                                                         |
| `vote_round_id`         | base64 (32 bytes)               | Vote round identifier.                                                     |
| `sighash`               | base64 (32 bytes)               | Client-computed sighash for signature verification.                        |

The server encodes these fields as a delegation transaction payload,
as specified in the "Delegation Transaction" section of
[^voting-protocol], and submits it to the vote chain.


### Sighash

The `sighash` field is the 32-byte ZIP 244 [^zip-244] shielded sighash
extracted from the signed PCZT after the hardware wallet signing flow.
Verifiers check the `spend_auth_sig` against this client-provided
sighash; nobody recomputes it. See [^orchard-balance-proof] for the
PCZT construction and signing flow that produces the sighash.

### Delegation Response

All transaction submission endpoints return the same response format:

```json
{
  "tx_hash": "<hex-encoded transaction hash>",
  "code": 0,
  "log": ""
}
```


| Field     | Type         | Description                                                             |
| --------- | ------------ | ----------------------------------------------------------------------- |
| `tx_hash` | string (hex) | Transaction hash for status polling.                                    |
| `code`    | uint32       | Result code. 0 indicates the transaction was accepted into the mempool. |
| `log`     | string       | Error description when `code` is non-zero. Omitted on success.          |


## Vote Commitment Transaction

The vote commitment transaction casts a vote on a specific proposal. It
corresponds to ZKP2 (the vote commitment circuit) as specified in
[^voting-protocol].

### Vote Commitment Endpoint

```
POST /shielded-vote/v1/cast-vote
```

### Vote Commitment Request Body

A JSON object with the following fields:


| Field                          | Type              | Description                                                   |
| ------------------------------ | ----------------- | ------------------------------------------------------------- |
| `van_nullifier`                | base64 (32 bytes) | Nullifier of the vote authority note being consumed.          |
| `vote_authority_note_new`      | base64 (32 bytes) | New vote authority note commitment.                           |
| `vote_commitment`              | base64 (32 bytes) | Vote commitment (Poseidon hash binding the vote).             |
| `proposal_id`                  | uint32            | Proposal identifier (1 to 50).                                |
| `proof`                        | base64 (variable) | Halo 2 ZKP2 proof.                                            |
| `vote_round_id`                | base64 (32 bytes) | Vote round identifier.                                        |
| `vote_comm_tree_anchor_height` | uint64            | Block height of the vote commitment tree root used as anchor. |
| `vct_root`                     | base64 (32 bytes) | The vote commitment tree root at the anchor height.           |
| `ea_pk`                        | base64 (32 bytes) | The round's election authority public key.                    |
| `vote_auth_sig`                | base64 (64 bytes) | RedPallas signature under the randomized voting key.          |
| `r_vpk`                        | base64 (32 bytes) | Randomized voting public key (compressed Pallas point).       |

The server encodes these fields as a vote transaction payload, as
specified in the "Vote Transaction" section of [^voting-protocol], and
submits it to the vote chain.


### Vote Commitment Response

Same response format as [Delegation Response].

## Share Submission

Share reveal takes place during the reveal window, after the round has
entered REVEALING and its VCT has been frozen. The wallet constructs
every Vote Reveal Proof itself, from material it retained when it cast
the vote, and submits every share reveal message itself; no other
party constructs a proof or holds a message before it is recorded. The
rules governing construction, independence of submissions and retry
are specified in the "Share Submission" section of [^voting-protocol].
This section specifies what the wallet keeps, the message it produces,
and the transport by which the message reaches the vote chain.

The wallet submits each share reveal message via [Direct Share Reveal]
when the vote chain reaches the height drawn for it (see
[Submission Timing]), over its own network path (see
[Network Isolation]). This requires the wallet to be running at that
height, in the foreground or in a background session; a wallet that
is not catches up as [Submission Timing] specifies. A voter MAY
instead choose express reveal for a vote, under which the wallet
submits all $N_s$ messages within one session; the conditions on
offering that choice are in [Submission Timing].

The wallet MUST NOT send any auxiliary input of the Vote Reveal Proof
— the vote commitment, its VCT position or path, the shares hash, the
share commitments, the blind factors, the vote decision, or a committed
ciphertext by itself — to any party, and MUST NOT hand a finished
message to any party other than a vote server for immediate
submission.

### Reveal Material Persistence

Because the final VCT root does not exist until voting closes, a wallet
cannot construct its Vote Reveal Proofs in the session in which it
votes. From the moment it constructs a vote commitment until every
share of that vote has been confirmed on the vote chain, a wallet MUST
persist, for that vote:

- the vote commitment and, once known, its VCT leaf position;
- `shares_hash` and all $N_s$ blinded share commitments;
- for each share $i$: the plaintext value $v_i$, the El Gamal
  randomness $r_i$, the blind factor $\mathsf{blind}_i$, and the
  committed ciphertext $(C_{1,i}, C_{2,i})$;
- the proposal identifier and the vote decision.

These values are defined in the "Vote Share" and "Vote Reveal Proof"
sections of [^voting-protocol]. This material MUST be stored encrypted
at rest and MUST NOT leave the device, in a backup or otherwise; in
particular it MUST NOT be sent to a vote server or PIR
endpoint. A wallet that loses
this material cannot reveal the vote, and the vote is not counted. A
wallet SHOULD zeroize the material once every share of the vote is
confirmed or the reveal window has closed.

### Share Reveal Message Format

The JSON encoding of a share reveal message, as defined in the "Share
Reveal Transaction" section of [^voting-protocol], is an object with
the following fields:

| Field                | Type                       | Description                                                          |
| -------------------- | -------------------------- | -------------------------------------------------------------------- |
| `proof`              | base64 (variable)          | Halo 2 Vote Reveal Proof.                                            |
| `share_nullifier`    | base64 (32 bytes)          | Share nullifier (see [Share Nullifier]).                             |
| `option_ciphertexts` | array of 8 objects         | Option-vector ciphertexts $E_0 \ldots E_{N_{\mathsf{opt}}-1}$, one per option position, each `{"c1", "c2"}`. |
| `proposal_id`        | uint32                     | Proposal identifier (1 to 50).                                       |
| `vct_root`           | base64 (32 bytes)          | The round's final VCT root.                                          |
| `vote_round_id`      | base64 (32 bytes)          | Vote round identifier.                                               |

Each `option_ciphertexts` entry contains:

| Field | Type              | Description                                                  |
| ----- | ----------------- | ------------------------------------------------------------ |
| `c1`  | base64 (32 bytes) | El Gamal ciphertext component `C1` (compressed Pallas point). |
| `c2`  | base64 (32 bytes) | El Gamal ciphertext component `C2` (compressed Pallas point). |

The array has exactly $N_{\mathsf{opt}} = 8$ entries, in option
position order, regardless of how many options the proposal has. A
share reveal message carries no signature, no submitter identity, no
share index, and no indication of which position holds the committed
ciphertext.

### Direct Share Reveal

```
POST /shielded-vote/v1/reveal-share
```

Submits one share reveal message to the vote chain. Served from a
`vote_servers` base URL. The server encodes the message as a share
reveal transaction payload, as specified in the "Share Reveal
Transaction" section of [^voting-protocol], and submits it to the vote
chain.

**Request body:** a [Share Reveal Message Format] object.

**Response:** the same response format as [Delegation Response].

A wallet MUST submit each message at its drawn height, or under the
catch-up rules (see [Submission Timing]), and over its own network
path (see [Network Isolation]). A wallet MAY use a different vote
server for each message, and MAY use a vote chain node of its own;
in the latter case the node's own connections to the network are
subject to [Network Isolation] as well, since the node becomes the
origin of every message it forwards.

### Share Nullifier

Each share has a deterministic nullifier derived from the vote
commitment, the share index and the share's blind factor. The wallet
computes this nullifier locally and hex-encodes it (lowercase, 64
characters) for use in the [Share Status] endpoint path. The
derivation is specified in [^voting-protocol].

### Share Status

```
GET /shielded-vote/v1/share-status/{roundId}/{nullifier}
```

Reports whether a share nullifier has been recorded on the vote chain.

**Path parameters:**

- `roundId`: Hex-encoded 32-byte vote round identifier (64 characters).
- `nullifier`: Hex-encoded 32-byte share nullifier (64 characters),
  computed as specified in [Share Nullifier].

**Response body:**

```json
{"status": "pending"}
```

| Field    | Type   | Description                                                                                       |
| -------- | ------ | ------------------------------------------------------------------------------------------------- |
| `status` | string | `"pending"` if not yet on-chain, `"confirmed"` if the share nullifier has been recorded on-chain. |

Querying a vote server for one's own share nullifiers
reveals to it which nullifiers are one's own; this is recorded as an
open issue in [^voting-protocol]. A wallet SHOULD therefore make status
polling optional and off by default. A wallet that does poll SHOULD
issue each query over a fresh network path (see [Network Isolation]),
SHOULD query at most one nullifier per path, and SHOULD do so at times
unrelated to the vote's submission schedule. A wallet MUST NOT require
a confirmed status before proceeding with any other step.

A wallet that observes, by whatever means, that a message has not been
recorded within a wallet-configured number of blocks MAY resubmit it
under the retry rules of [^voting-protocol], to a different vote
server and over a fresh network path. A duplicate that is recorded is
not effective, by its nullifier, and is harmless.

## Vote Commitment Tree

The vote commitment tree is an append-only Merkle tree that records
vote authority notes and vote commitments.

The vote commitment tree structure, hash function, and leaf encoding
are specified in [^voting-protocol]. Wallet clients interact with the
tree through the query endpoints defined in [Commitment Tree (Latest)],
[Commitment Tree at Height], and [Commitment Tree Leaves].

## Nullifier Exclusion Proof Retrieval

A nullifier exclusion proof is a Merkle non-membership proof for a
note's standard Orchard nullifier against the snapshot's nullifier
non-membership tree, whose construction is specified in
[^orchard-balance-proof].

A wallet that holds the set of Orchard nullifiers revealed at or before
`snapshot_height` — for example because it has scanned the chain over
that range — constructs the proof itself and contacts no server. This
is the RECOMMENDED path: asking a server for the proof reveals which
nullifier was asked about, and therefore which note the wallet holds,
to a party that learns nothing otherwise.

A wallet that does not hold the set MAY retrieve the proof from one of
the `pir_endpoints` published in the vote configuration. The retrieval
protocol such a server offers, and whether it conceals the queried
nullifier from the server, are properties of the deployment and are not
specified here; version selection follows [Version Handling]. A wallet
using this path MUST NOT present it to the user as equivalent in
privacy to constructing the proof locally, unless the server's protocol
conceals the query.

## Client Privacy Requirements

The protocol's privacy claims depend on client behaviour that no
server-side specification can enforce. This section states that
behaviour normatively, so that a wallet's conformance is a checkable
property rather than an implementation preference.

### Network Isolation

A wallet MUST route each share reveal message over a network path that
is not shared with any other message of the same vote — for example, a
fresh Tor circuit or mixnet channel per message. This MUST be the
default behaviour, not an opt-in setting.

A wallet MUST use the same protection for the requests that precede
voting and reveal, in particular commitment tree synchronisation and
PIR queries, and MUST NOT make any request to a vote server or PIR
endpoint over a path that has carried the wallet's ordinary Zcash
light client traffic for the same user.

The reason for the second requirement is compositional. The voting
layer exposes an association between a network identity and a vote
weight. A Zcash light client exposes an association between a network
identity and a set of addresses. Neither is individually sufficient to
link an address to a balance; together they are. A wallet that protects
one and not the other has protected neither.

A wallet that cannot satisfy these requirements MUST inform the user
before the vote is cast, rather than proceeding silently.

### Submission Timing

A wallet MUST construct a submission schedule within the reveal
window, against `reveal_end_height`, as specified in the "Submission
Timing" section of [^voting-protocol], which adopts the scheduling
discipline ZIP 318 [^zip-0318] defines for pool-crossing transfers.
The schedule is drawn in vote chain block heights; the wallet uses the
configuration's `block_time_seconds` to present heights to the user as
times. In summary, and normatively by reference to that document, a
wallet:

- MUST shuffle the vote's shares into a uniformly random order before
  assigning submission heights, so that the order in which share values
  are emitted does not depend on their magnitudes or their indices;
- MUST draw each successive inter-submission delay independently from
  an exponential distribution, so that its submissions approximate a
  Poisson process, rather than spacing them evenly or by a fixed
  interval;
- MUST draw all such randomness from a cryptographically secure random
  number generator;
- MUST NOT submit a vote's shares as a single batch;
- SHOULD construct and persist all $N_s$ messages when it draws the
  schedule, so that each later submission is a single request with no
  synchronisation.

A wallet MUST NOT place a voter's entire ballot count into a single
share.

**Background submission.** Where the platform grants background
execution, the wallet SHOULD submit each message from a background
session at its drawn height. A background session that submits a share
reveal message MUST NOT also synchronise the wallet's Zcash state or
make any other request that identifies the wallet.

**Catching up.** On every application open during the reveal window,
the wallet MUST reconcile its schedule against what the chain has
recorded and MUST surface any overdue message. It MUST NOT submit more
than one overdue message in that session; the rest are rescheduled by
redrawing their delays from the current height, and the wallet SHOULD
tell the user that the vote is not yet fully revealed and when to
return. Reconciliation on open MUST NOT rely on notification delivery.

**When the window is short.** Where insufficient blocks remain before
`reveal_end_height`, less the safety margin $\Delta$ that
[^voting-protocol] specifies, to run the full schedule, a wallet MUST
draw each remaining share's submission height independently and
uniformly from the remaining interval, and MUST NOT submit the
remaining shares together.
Submitting promptly is not a substitute for submitting independently: a
wallet that responds to a closing round by sending everything at once
reproduces through timing precisely the exposure that removing
single-share mode was intended to prevent.

Where the remaining window is too short to submit all shares even under
the compressed schedule, a wallet MUST inform the voter before
proceeding rather than submitting silently.

**Express reveal.** A wallet MAY offer the voter, for each vote, the
choice of express reveal as specified in the "Share Submission" and
"Submission Timing" sections of [^voting-protocol]: all $N_s$ messages
submitted within the current session, each at a height drawn uniformly
from a short interval, each over its own network path. A wallet that
offers it:

- MUST leave it off by default, and MUST NOT remember a previous
  choice as the default for a later vote;
- MUST obtain the voter's choice for the specific vote, before
  submitting the first message of that vote;
- MUST, before the voter chooses, state that choosing express reveal
  lets any party holding $t$ of the round's key shares learn this
  vote's weight and choice, and lets anyone reading the vote chain see
  that these shares belong to one vote, and MUST state what the
  alternative requires of the voter (returning across the reveal
  window, or leaving the wallet able to run in the background);
- MUST NOT describe express reveal as private, fast-and-private, or in
  any terms that omit the cost;
- MUST NOT offer, under the name of express reveal or any other, a
  mode that reduces the share count, places the ballot count in one
  share, exposes the decision, or sends any message or witness
  material to a party other than a vote server for immediate
  submission;
- MUST apply [Network Isolation] to every message of an express
  reveal.

A wallet MAY offer express reveal when the remaining window is short,
alongside the compressed schedule above, but MUST NOT select it on the
voter's behalf in that case either.

**Returning to reveal.** A vote is counted only if the wallet is opened
during the reveal window, obtains the final VCT root, constructs its
share reveal messages and submits them, across the window or, under
express reveal, within that session. A wallet that is not opened
between `vote_end_height` and `reveal_end_height` cannot reveal, and
the vote is lost; one that is opened only once, on a platform with no
background execution and without express reveal, reveals only part of
it. A wallet SHOULD surface this to the user when the vote is cast,
SHOULD present both deadlines as estimated times, and SHOULD prompt
the user to return during the reveal window, early enough that the
short-window case above is avoidable, since every option available
once the window is short is worse than having started sooner.

### Partial Delegation

A wallet SHOULD allow a voter to delegate part of their balance rather
than all of it, and SHOULD default to presenting this choice rather
than delegating the full balance implicitly.

A voter who delegates their entire balance exposes that entire balance
to whatever residual disclosure the protocol permits. A voter who
delegates a chosen amount exposes only that amount. Since the delegated
quantity is the quantity at risk, the choice belongs to the voter.

Where the underlying protocol does not yet specify a partial delegation
transaction, a wallet SHOULD allow the voter to delegate a subset of
their notes, which achieves a coarser form of the same control.


## Version Handling

All version strings in `supported_versions` use the form `"v" MAJOR`
(e.g., `"v0"`, `"v1"`). A major version bump indicates a
breaking change; within the same major version, implementations remain
compatible.

A wallet MUST reject the configuration if it does not support the
advertised `vote_server`, `vote_protocol`, or `tally` version, or if
`pir` contains no version it supports. If any check fails, the wallet
MUST NOT proceed and SHOULD prompt the user to update.

### URL Path Prefix

REST endpoint paths include a version prefix (e.g.,
`/shielded-vote/v1/`). The path version corresponds to the major
version declared in `supported_versions.vote_server`. A `vote_server`
value of `"v1"` uses the `/shielded-vote/v1/` prefix.

### Relationship to `config_version`

`config_version` versions the structure of the vote configuration JSON
document itself (field names, types, nesting). The component versions
(`vote_server`, `vote_protocol`, `tally`, `pir`) version protocol
behavior. These can all evolve independently: a structural change to
the config schema (e.g., adding a new required top-level field) bumps
`config_version`, while a change to endpoint behavior bumps
`vote_server`, a change to circuits or tree structure bumps
`vote_protocol`, and a change to decryption or aggregation bumps
`tally`.

`config_version` also selects the version in the poll signature's
domain separator (see [Configuration Authentication]). Version 4, the
version this specification defines, is verified under
`ZcashVotingPollSignature:v4`.

## Transaction Lifecycle

### Broadcast Semantics

Transaction submission endpoints (`/delegate-vote`, `/cast-vote`,
`/reveal-share`) return synchronously. A successful response (HTTP
200, `code` = 0) indicates that the transaction entered the vote
server's mempool. It does not guarantee that it will be recorded, and
being recorded does not make it effective: the chain applies none of
the protocol's rules (see the "Vote Chain Record" section of
[^voting-protocol]). A vote server MAY check a transaction against
those rules before accepting it, as a service to the wallet, and
SHOULD say so in `log` when it rejects one on that basis; a wallet
MUST NOT treat such a rejection as final, since another node may
record the transaction.

### Confirmation Polling

After receiving a successful broadcast response, the wallet SHOULD poll
the [Transaction Status] endpoint using the `tx_hash` from the response.
The transaction is confirmed when the response includes a non-empty
`height` and `code` = 0.

### Timeouts

A vote server that verifies zero-knowledge proofs before accepting a
transaction may take tens of seconds to respond. Wallet HTTP clients
SHOULD use a timeout of at least 120 seconds for transaction
submission requests.

## Encoding Conventions

All data exchanged between wallet clients and servers uses the encodings
described in this section.

### Cryptographic Types


| Type                      | Size     | Encoding                                                                                                     |
| ------------------------- | -------- | ------------------------------------------------------------------------------------------------------------ |
| Pallas base field element | 32 bytes | Little-endian canonical representation. Implementations MUST reject values >= the Pallas base field modulus. |
| Compressed Pallas point   | 32 bytes | Standard Pallas point compression. [^protocol]                                                               |
| ElGamal ciphertext        | 64 bytes | `C1` (32 bytes) followed by `C2` (32 bytes), each a compressed Pallas point.                                 |
| Halo 2 proof              | variable | Opaque byte sequence.                                                                                        |
| RedPallas signature       | 64 bytes | `R` (32 bytes) followed by `s` (32 bytes).                                                                   |
| Ed25519 public key        | 32 bytes | As specified in RFC 8032 [^rfc8032].                                                                         |
| Ed25519 signature         | 64 bytes | As specified in RFC 8032 [^rfc8032].                                                                         |
| Transaction payload       | variable | The byte encoding specified in the "Transaction Formats" section of [^voting-protocol].                       |


### JSON Encoding

All REST endpoints accept and return `application/json`.

- **Byte arrays**: Standard base64 encoding with padding (RFC 4648
Section 4 [^rfc4648]).
- **`vote_round_id`**: The encoding of `vote_round_id` varies by
context. The following table lists every occurrence and its encoding:

| Context                                         | Encoding                      |
| ----------------------------------------------- | ----------------------------- |
| Vote configuration JSON (`vote_round_id` field) | Hex (64 lowercase characters) |
| URL path parameters (`{round_id}`, `{roundId}`) | Hex (64 lowercase characters) |
| Delegation request body (`vote_round_id`)       | Base64 (32 bytes)             |
| Vote commitment request body (`vote_round_id`)  | Base64 (32 bytes)             |
| Share reveal message (`vote_round_id`)          | Base64 (32 bytes)             |
| Chain query response bodies (`vote_round_id`)   | Base64 (32 bytes)             |
- **Integers**: JSON numbers. Fields typed `uint32` or `uint64` in the
protocol definition are encoded as JSON numbers.
- **Enumerations**: JSON numbers corresponding to the protobuf enum
value (e.g., `SESSION_STATUS_ACTIVE` = 1).

# Deployment

A wallet implementation MUST record, and SHOULD make visible to the
user or in release documentation, the versions of the components it
was built against:

| Component | Why it is pinned |
|---|---|
| Vote protocol circuits | Determine what the proofs a wallet constructs actually prove. |
| Vote chain / SDK | Determines transaction acceptance and chain semantics. |
| Client voting library | Determines share decomposition and submission timing behaviour. |

Recording the client library version alone is insufficient: the
circuits determine the meaning of the proofs, and a library version
does not identify them.

Where a wallet implements share decomposition or submission timing
itself rather than consuming them from a shared
library, it MUST state this, because such a wallet does not inherit
changes to those behaviours when the library is updated.


# Rationale

## Unified Vote Servers

All chain endpoints — queries and transaction submission, including
direct share reveal — are served under the `/shielded-vote/v1/` path
prefix from the same `vote_servers` base URLs. In the current
architecture, a single `svoted` process hosts every chain endpoint on
the same port. Every endpoint is a convenience over the vote chain's
record: a wallet with its own vote chain node needs none of them, and
a wallet using them is trusting the server's derivation only as far as
[Binding to the Chain Round] allows.

## Independent Component Versions

Each `supported_versions` field tracks a component that can change on
its own schedule: `vote_server` covers the REST API surface,
`vote_protocol` covers the ZKP circuits and commitment tree structure,
`tally` covers threshold decryption and result aggregation, and `pir`
covers nullifier retrieval. Separating these avoids forcing a
wallet update when only one component changes — for example, a new
tally method does not require wallets to update their proof generation
code.

## JSON over Protobuf

The REST API uses JSON encoding rather than binary protobuf serialization.
JSON is broadly supported across wallet development stacks (Swift, Kotlin,
TypeScript, Rust) and does not require protobuf code generation. The
trade-off in message size is acceptable for the request and response
volumes involved.

# Open issues

- [Share Status] lets a wallet confirm inclusion only by disclosing
  which nullifiers are its own. A private-retrieval confirmation
  mechanism is an open issue in [^voting-protocol].
- `block_time_seconds` is unsigned and informational, and the chain's
  observed interval drifts from the published target. A wallet that
  sizes its submission schedule from a wrong value spreads its
  submissions badly; a wallet SHOULD check it against the interval it
  observes between vote chain blocks, but no endpoint exposes block
  timestamps yet.
- [Snapshot Verification] requires the wallet's Zcash consensus node or
  light client backend to serve the nullifier non-membership tree root
  at a given height. No Zcash node or light client API for this is
  specified; it is a Zcash-node-side interface outside this document,
  and until one exists a wallet cannot complete the verification and
  so cannot take part in a round.


# Reference implementation

No implementation conforming to this specification is available at
the time of writing. A vote chain with a REST API that is expected to
be adapted to it is at
[valargroup/vote-sdk](https://github.com/valargroup/vote-sdk).

# References

[^BCP14]: [Information on BCP 14 -- "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^rfc4648]: [RFC 4648: The Base16, Base32, and Base64 Data Encodings](https://www.rfc-editor.org/rfc/rfc4648)

[^zip-0318]: [ZIP 318: Orchard to Ironwood Migration](zip-0318.md)

[^voting-protocol]: [Draft ZIP: Shielded Voting Protocol](draft-valargroup-shielded-voting.md)

[^voting-setup]: [Draft ZIP: Zcash Shielded Coinholder Voting](draft-valargroup-shielded-voting-setup.md)

[^rfc8032]: [RFC 8032: Edwards-Curve Digital Signature Algorithm (EdDSA)](https://www.rfc-editor.org/rfc/rfc8032)


[^orchard-balance-proof]: [Draft ZIP: Orchard Proof-of-Balance](draft-valargroup-orchard-balance-proof.md)

[^zip-244]: [ZIP 244: Transaction Identifier Non-Malleability](zip-0244.rst)
