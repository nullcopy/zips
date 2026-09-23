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

Delegation

: The act of proving ownership of unspent Orchard notes in the Ironwood
  pool at the snapshot height and registering a vote authority note on
  the vote commitment tree. See [^orchard-balance-proof].

Ironwood pool

: The Zcash shielded pool over which votes are weighted. The Ironwood
  pool uses the Orchard protocol; references in this document to
  Orchard keys, notes, nullifiers, signatures or circuits refer to
  those constructions as used in the Ironwood pool. See
  [^voting-protocol].

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

Relay

: An untrusted store-and-forward service to which a wallet MAY hand a
  finished share reveal message for submission to the vote chain at a
  requested time. A relay constructs no proofs and receives no witness
  material. See [Share Submission] and the "Share Submission" section
  of [^voting-protocol].

Reveal window

: The period, following the voting window, during which the vote chain
  accepts share reveal transactions for a round. It ends at the round's
  `reveal_end_time`. See the "Round Lifecycle" section of
  [^voting-protocol].

Share

: One of the $N_s$ encrypted fragments of the holder's ballot count
  within a vote commitment. Each share is revealed independently
  during the reveal window by a share reveal message that the wallet
  constructs itself.

Share reveal message

: The message a share reveal transaction carries: a Vote Reveal Proof,
  a share nullifier, the option-vector ciphertexts, the proposal
  identifier, the final VCT root and the round identifier. Specified in
  the "Share Reveal Message" section of [^voting-protocol]; its JSON
  encoding is given in [Share Reveal Message Format].

Trustee

: One of the parties named in a round's configuration that jointly
  generate the round's election authority key and each hold a share of
  its private key. Every trustee ratifies the round by acknowledging
  that key on the vote chain; a wallet verifies those acknowledgements
  before encrypting to it. See [Binding to the Chain Round] and the
  "Election Authority Key Ceremony" section of [^voting-protocol].

Vote authority note (VAN)

: A note appended to the vote commitment tree during delegation. The VAN
  carries the delegated vote weight and is consumed (nullified) when
  the holder casts a vote.

Vote commitment tree

: An append-only Merkle tree that records vote authority notes and vote
  commitments. The tree root at a given block height serves as a public
  input to zero-knowledge proof verification.

Vote round

: A time-bounded voting session defining a set of proposals, a Zcash
  snapshot height, a voting deadline and a reveal deadline. Wallet
  clients interact with exactly one vote round at a time. The round
  states and their transitions are specified in the "Round Lifecycle"
  section of [^voting-protocol].

# Abstract

This ZIP specifies the REST API endpoints, wire formats, and discovery
mechanism that wallet clients use to participate in shielded on-chain
voting rounds. It covers vote round discovery via a per-vote
configuration document signed by the round's poll runner, the checks by
which a wallet verifies a round's snapshot roots and election authority
key for itself, data query endpoints for reading chain state,
transaction submission endpoints for delegation, vote casting and share
reveal, the payload a wallet hands to a relay, and the encoding
conventions for all exchanged data.

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
backwards compatibility with deployed wallets.

# Requirements

- A wallet can discover and join an active voting round using a published
configuration document.
- A wallet can submit delegation and vote commitment transactions
using the wire formats in this specification. Proof construction is
specified in companion ZIPs.
- A wallet can submit share reveal transactions directly, or hand
finished share reveal messages to relays, without disclosing to any
third party which shares belong to the same vote or which option the
vote supports.
- A wallet retains, from the moment it casts a vote until every share
of that vote is revealed, the material the Vote Reveal Proofs need,
and that material never leaves the device.
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

- Trustee onboarding, key registration, and the EA key ceremony
(distributed key generation), specified in [^voting-protocol] and
[^voting-setup].
- Chain consensus rules and block production.
- Round creation and the poll runner's operations.
- The internal implementation of relays. A relay's obligations are
specified in [^voting-protocol]; this document specifies only the
payload a wallet hands to one.

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
at `vote_end_time`.

## Share Reveal (during REVEALING)

The reveal window opens at `vote_end_time` and closes at
`reveal_end_time`. Share reveal messages cannot be constructed before
it opens, because they are anchored to the round's final VCT root,
which exists only once voting has closed. A wallet that is not opened
at least once during the reveal window cannot reveal its shares, and
the vote is not counted. [Submission Timing] requires wallets to
surface this to the user.

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

17. **Draw the submission schedule.** Draw $N_s$ submission times
    within the reveal window as specified in [Submission Timing].

18. **Submit.** Either submit each message directly via
    `POST /shielded-vote/v1/reveal-share` at its scheduled time, or
    hand each finished message, with its scheduled time as
    `submit_at`, to a distinct relay via `POST /shielded-vote/v1/shares`.
    Each submission or hand-off uses its own network path. See
    [Share Submission] and [Network Isolation].

19. **Optionally confirm inclusion.** A wallet MAY check
    `GET /shielded-vote/v1/share-status/{roundId}/{nullifier}`, subject
    to the privacy caveats in [Share Status]. This is not a required
    step.

## Results (optional)

20. **View tally results.** After the round reaches FINALIZED status,
    query `GET /shielded-vote/v1/tally-results/{round_id}` for
    decrypted per-proposal tallies. See [Tally Results].

# Specification

## Vote Discovery

A vote configuration is a JSON document published for each vote round.
It contains all parameters a wallet needs to locate services and
participate in the round.

### Vote Configuration Format

```json
{
  "config_version": 3,
  "vote_round_id": "<hex, 64 characters>",
  "vote_servers": [
    {"url": "https://vote1.example.com", "label": "validator-1"}
  ],
  "relays": [
    {"url": "https://relay1.example.com", "label": "relay-1"},
    {"url": "https://relay2.example.com", "label": "relay-2"}
  ],
  "pir_endpoints": [
    {"url": "https://pir1.example.com", "label": "pir-1"}
  ],
  "snapshot_height": 2800000,
  "snapshot_blockhash": "<base64, 32 bytes>",
  "nc_root": "<base64, 32 bytes>",
  "nullifier_imt_root": "<base64, 32 bytes>",
  "vote_end_time": 1735689600,
  "reveal_end_time": 1736294400,
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
    {
      "label": "trustee-1",
      "address": "<vote chain account address>",
      "account_pk": "<base64, 32 bytes>"
    },
    {
      "label": "trustee-2",
      "address": "<vote chain account address>",
      "account_pk": "<base64, 32 bytes>"
    }
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
| `config_version`                   | integer          | Schema version of this configuration document. This specification defines version 3.                                           |
| `vote_round_id`                    | string           | Hex-encoded 32-byte vote round identifier (64 characters, lowercase).                                                          |
| `vote_servers`                     | array            | One or more vote server base URLs serving the chain query and transaction submission endpoints. Each entry has `url` (string) and `label` (string). |
| `relays`                           | array            | Zero or more relay base URLs serving the [Relay Hand-off] endpoint. Each entry has `url` (string) and `label` (string). See [Share Submission]. |
| `pir_endpoints`                    | array            | One or more nullifier PIR server base URLs. Each entry has `url` and `label`.                                                  |
| `snapshot_height`                  | integer          | Zcash block height at which the Ironwood pool snapshot was taken.                                                              |
| `snapshot_blockhash`               | string           | Base64-encoded 32-byte hash of the Zcash block at `snapshot_height`.                                                           |
| `nc_root`                          | string           | Base64-encoded 32-byte Ironwood pool note commitment tree root at the snapshot.                                                |
| `nullifier_imt_root`               | string           | Base64-encoded 32-byte nullifier non-membership tree root at the snapshot.                                                     |
| `vote_end_time`                    | integer          | Unix timestamp (seconds) after which votes are no longer accepted; the round enters REVEALING.                                 |
| `reveal_end_time`                  | integer          | Unix timestamp (seconds) after which share reveals are no longer accepted; the round enters TALLYING.                          |
| `proposals`                        | array            | Ordered list of proposals. Each has `id` (integer, 1-indexed), `title` (string), `description` (string), and `options` (array of `{index, label}`). |
| `trustees`                         | array            | The round's trustees, in the order hashed by [Trustees Hash]. Each entry has `label` (string), `address` (string, the trustee's vote chain account address) and `account_pk` (base64, the 32-byte ed25519 account public key under which the trustee signs its chain transactions). |
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
specification defines version 3. A version 1 document carries no
signature, and a version 2 document carries neither `reveal_end_time`
nor `trustees` and is signed under a model this specification does not
define: a wallet MUST NOT accept either for a round created after this
specification takes effect. A wallet that accepts a version 1 or 2
document for an earlier round MUST NOT present that round to the user
as authenticated.
- `vote_round_id` MUST be exactly 64 lowercase hexadecimal characters.
- `vote_servers` MUST contain at least one entry.
- `relays`, if present, MUST be an array; each entry MUST have a
string `url` and a string `label`. It MAY be empty. A wallet MUST
treat an absent `relays` field as an empty array.
- `pir_endpoints` MUST contain at least one entry.
- `snapshot_height` MUST be greater than 0.
- `reveal_end_time` MUST be greater than `vote_end_time`. The minimum
separation the chain enforces is specified in the "Round Lifecycle"
section of [^voting-protocol].
- `snapshot_blockhash`, `nc_root` and `nullifier_imt_root` MUST each be
the base64 encoding of exactly 32 bytes.
- `trustees` MUST contain at least 2 entries. Each entry MUST have a
string `label`, a string `address`, and an `account_pk` that is the
base64 encoding of exactly 32 bytes. `address` values MUST be unique
across entries.
- `poll_signature` MUST be an object with a string `key_id`, a string
`alg`, and a base64 `sig`.
- `proposals` MUST contain between 1 and 15 entries.
- Each proposal MUST have between 2 and 8 options.
- Proposal `id` values MUST be unique and in the range 1 to 15.
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
   `ZcashVotingPollSignature:v3` (27 bytes), then `vote_round_id`
   (32 bytes, decoded from hex), `snapshot_height` (4 bytes,
   big-endian unsigned), `snapshot_blockhash`, `nc_root` and
   `nullifier_imt_root` (32 bytes each, decoded from base64),
   `proposals_hash` (32 bytes, computed from the configuration's
   `proposals` per [Proposals Hash]), `vote_end_time` and
   `reveal_end_time` (8 bytes each, big-endian unsigned), and
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
`ea_pk`, which does not exist when the poll runner signs, nor
`min_confirmations`, which the wallet does not check; a configuration
can therefore be signed and verified before the round opens.

### Trustees Hash

`trustees_hash` is the 32-byte BLAKE2b-256 hash, with personalization
`ZcashVoteTrustee` (16 bytes), of the concatenation of the `account_pk` values
of the configuration's `trustees` entries, each decoded from base64 to
its 32 bytes, in array order and without length prefixes. For $n$
trustees the input is exactly $32n$ bytes. Neither `label` nor
`address` enters the hash: the poll signature binds the trustees by
their keys, and a wallet identifies a trustee's acknowledgement by its
`account_pk`, not by its label.

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
`nc_root`, `nullifier_imt_root`, `vote_end_time` and `reveal_end_time`
as the authenticated configuration, and that its `proposals_hash`
equals the hash of the configuration's `proposals` computed per
[Proposals Hash]. A wallet MUST NOT take part in a round that fails
this check.

**Election authority key.** `ea_pk` is derived by the vote chain when
the key ceremony completes and is not part of the configuration; the
wallet reads it from the `VoteRound`. Before taking part in a round,
and in any case before encrypting anything to `ea_pk`, a wallet MUST
fetch the round's acknowledgement transactions via
[Round Acknowledgements] and verify that for every entry of the
configuration's `trustees` there is an acknowledgement such that:

1. `signer_pk` equals that trustee's `account_pk` and
   `trustee_address` equals that trustee's `address`;
2. `payload` equals SHA-256 over the concatenation of the ASCII string
   `ack`, `vote_round_id` (32 bytes), the round's `ea_pk` (32 bytes)
   and the trustee's `address` (its raw account address bytes, without
   human-readable prefix or checksum), as
   specified in the "Election Authority Key Ceremony" section of
   [^voting-protocol];
3. `signature` is a valid ed25519 signature [^rfc8032] by `signer_pk`
   over `payload`.

A wallet MUST NOT encrypt to an `ea_pk` for which this check fails for
any trustee, and MUST NOT take part in the round. An `ea_pk`
acknowledged by every trustee under its own account key is the key
those trustees hold shares of; see the "Poll Signature" and
"Ratification" sections of [^voting-protocol].

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

Returns the active voting round, if any.

**Response body:** A JSON object containing a `round` field with the
`VoteRound` structure:


| Field                | Type              | Description                                                          |
| -------------------- | ----------------- | -------------------------------------------------------------------- |
| `vote_round_id`      | base64 (32 bytes) | Round identifier.                                                    |
| `snapshot_height`    | uint64            | Zcash snapshot block height.                                         |
| `snapshot_blockhash` | base64 (32 bytes) | Zcash block hash at snapshot.                                        |
| `proposals_hash`     | base64 (32 bytes) | SHA-256 hash of the proposals array (see [Proposals Hash]).          |
| `vote_end_time`      | uint64            | Unix timestamp (seconds) at which voting closes and the reveal window opens. |
| `reveal_end_time`    | uint64            | Unix timestamp (seconds) at which the reveal window closes.          |
| `nullifier_imt_root` | base64 (32 bytes) | Nullifier non-membership tree root.                                  |
| `nc_root`            | base64 (32 bytes) | Ironwood pool note commitment tree root.                             |
| `status`             | uint32            | Session status enum (4=PENDING, 1=ACTIVE, 5=REVEALING, 2=TALLYING, 3=FINALIZED); see the "Round Lifecycle" section of [^voting-protocol]. |
| `ea_pk`              | base64 (32 bytes) | Election authority public key (compressed Pallas point).             |
| `proposals`          | array             | Proposals with `id` (uint32), `title`, `description`, and `options`. |
| `description`        | string            | Human-readable round description.                                    |
| `title`              | string            | Short human-readable round title.                                    |
| `creator`            | string            | Address of the account that created the session.                     |
| `created_at_height`  | uint64            | Vote chain block height at which the round was created.              |
| `trustees`           | array             | Trustees named in the round creation transaction. Informational: a wallet binds to the configuration's `trustees`, not to this field. |
| `min_confirmations`  | uint64            | Confirmation depth the poll runner applied when choosing the snapshot. Informational; not checked by the wallet. |


The response may contain additional fields related to the EA key
ceremony and threshold decryption (e.g., ceremony status, trustee
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

The `proposals_hash` is SHA-256 over the canonical JSON serialization
of the `proposals` array, allowing wallets to verify that the
proposals shown to the user match those on the vote chain.

To compute: construct a JSON array of proposal objects containing
`id`, `title`, `description`, and `options` (each with `index`, `label`),
ordered by `id` then `index` ascending, with object keys in the order
listed. Serialize with no whitespace and SHA-256 the UTF-8 result. The
JSON serialization MUST NOT escape the forward slash character (`/`);
non-ASCII characters are emitted as UTF-8 bytes, not as `\uXXXX`
escapes. Control characters (U+0000 through U+001F) are escaped per
RFC 8259 (`\b`, `\f`, `\n`, `\r`, `\t`, or `\u00XX`).

Example preimage (from [Vote Configuration Format]):

```
[{"id":1,"title":"Approve protocol upgrade","description":"Approve or oppose the proposed protocol upgrade.","options":[{"index":0,"label":"Support"},{"index":1,"label":"Oppose"}]}]
```

SHA-256 of the above preimage: `3f9a361d43c4ddb77ad138a091374e2e2958718e64937f33df99a09bd567e63d`.

Wallets SHOULD verify `proposals_hash` against the vote
configuration before proceeding.

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
array. Each entry represents one acknowledgement transaction:

| Field             | Type              | Description                                                                 |
| ----------------- | ----------------- | --------------------------------------------------------------------------- |
| `trustee_address` | string            | Vote chain account address of the acknowledging trustee.                    |
| `payload`         | hex (32 bytes)    | The SHA-256 acknowledgement payload, as specified in [Binding to the Chain Round]. |
| `signature`       | base64 (64 bytes) | ed25519 signature over `payload` by `signer_pk`.                            |
| `signer_pk`       | base64 (32 bytes) | ed25519 account public key of the signer.                                   |

The array is empty for a round no trustee has yet acknowledged. The
server does not verify entries on the wallet's behalf: the wallet MUST
apply the checks in [Binding to the Chain Round] to each entry and MUST
NOT infer anything from an entry's presence alone.

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

Returns finalized (decrypted) tally results for a vote round. Only
available after the round reaches FINALIZED status.

**Path parameters:**

- `round_id`: Hex-encoded 32-byte vote round identifier.

**Response body:** A JSON object containing a `results` array:


| Field           | Type              | Description                          |
| --------------- | ----------------- | ------------------------------------ |
| `vote_round_id` | base64 (32 bytes) | Round identifier.                    |
| `proposal_id`   | uint32            | Proposal identifier.                 |
| `vote_decision` | uint32            | Option position whose aggregate this entry reports. It is a property of the aggregate, not of any voter. |
| `total_value`   | uint64            | Decrypted aggregate value (zatoshi). |


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
| `height` | string | Block height at which the transaction was included (empty if pending). |
| `code`   | uint32 | Result code (0 = success).                                             |
| `log`    | string | Error message if `code` is non-zero.                                   |
| `events` | array  | ABCI events emitted by the transaction.                                |


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
| `gov_nullifiers`        | array of base64 (32 bytes each) | Governance nullifiers (up to 5). One per claimed note.                     |
| `proof`                 | base64 (variable)               | Halo 2 ZKP1 proof.                                                         |
| `vote_round_id`         | base64 (32 bytes)               | Vote round identifier.                                                     |
| `sighash`               | base64 (32 bytes)               | Client-computed sighash for signature verification.                        |


### Sighash

The `sighash` field is the 32-byte ZIP 244 [^zip-244] shielded sighash
extracted from the signed PCZT after the hardware wallet signing flow.
The chain verifies the `spend_auth_sig` against this client-provided
sighash; it does not recompute it. See [^orchard-balance-proof] for the
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
| `proposal_id`                  | uint32            | Proposal identifier (1 to 15).                                |
| `proof`                        | base64 (variable) | Halo 2 ZKP2 proof.                                            |
| `vote_round_id`                | base64 (32 bytes) | Vote round identifier.                                        |
| `vote_comm_tree_anchor_height` | uint64            | Block height of the vote commitment tree root used as anchor. |
| `vote_auth_sig`                | base64 (64 bytes) | RedPallas signature under the randomized voting key.          |
| `r_vpk`                        | base64 (32 bytes) | Randomized voting public key (compressed Pallas point).       |


### Vote Commitment Response

Same response format as [Delegation Response].

## Share Submission

Share reveal takes place during the reveal window, after the round has
entered REVEALING and its VCT has been frozen. The wallet constructs
every Vote Reveal Proof itself, from material it retained when it cast
the vote; no other party constructs one. The rules governing
construction, independence of submissions, relay selection and retry
are specified in the "Share Submission" section of [^voting-protocol].
This section specifies what the wallet keeps, the message it produces,
and the two transports by which the message reaches the vote chain.

**Direct submission.** The wallet submits each share reveal message
itself, at its scheduled time, via [Direct Share Reveal]. This requires
the wallet to be online at each scheduled time.

**Relayed submission.** A wallet that will not be online for the
duration of its schedule MAY hand each finished message, together with
its scheduled time, to a relay via [Relay Hand-off]. The relay holds
exactly what the chain will hold and learns from the payload nothing a
chain observer would not.

On either path the wallet MUST NOT send any auxiliary input of the Vote
Reveal Proof — the vote commitment, its VCT position or path, the
shares hash, the share commitments, the blind factors, the vote
decision, or a committed ciphertext by itself — to any party.

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
particular it MUST NOT be sent to a vote server, relay or PIR
endpoint. A wallet that loses
this material cannot reveal the vote, and the vote is not counted. A
wallet SHOULD zeroize the material once every share of the vote is
confirmed or the reveal window has closed.

### Share Reveal Message Format

The JSON encoding of a share reveal message, as defined in the "Share
Reveal Message" section of [^voting-protocol], is an object with the
following fields:

| Field                | Type                       | Description                                                          |
| -------------------- | -------------------------- | -------------------------------------------------------------------- |
| `proof`              | base64 (variable)          | Halo 2 Vote Reveal Proof.                                            |
| `share_nullifier`    | base64 (32 bytes)          | Share nullifier (see [Share Nullifier]).                             |
| `option_ciphertexts` | array of 8 objects         | Option-vector ciphertexts $E_0 \ldots E_{N_{\mathsf{opt}}-1}$, one per option position, each `{"c1", "c2"}`. |
| `proposal_id`        | uint32                     | Proposal identifier (1 to 15).                                       |
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

Submits one share reveal message directly to the vote chain. Served
from a `vote_servers` base URL.

**Request body:** a [Share Reveal Message Format] object.

**Response:** the same response format as [Delegation Response].

A wallet submitting directly MUST submit each message at its scheduled
time (see [Submission Timing]) and over its own network path (see
[Network Isolation]).

### Relay Hand-off

```
POST /shielded-vote/v1/shares
```

Hands one finished share reveal message to a relay for submission at a
requested time. Served from a `relays` base URL, not from a vote
server.

**Request body:** A JSON object with the following fields:

| Field       | Type   | Description                                                                                                        |
| ----------- | ------ | ------------------------------------------------------------------------------------------------------------------ |
| `message`   | object | A [Share Reveal Message Format] object, complete and unaltered.                                                    |
| `submit_at` | uint64 | Unix timestamp (seconds) at which the relay is to submit the message. 0 means as soon as possible.                 |

The payload MUST NOT contain anything else. In particular it MUST NOT
contain the vote commitment, its VCT position, the shares hash, the
share commitments, any blind factor, the vote decision, the share
index, or the committed ciphertext other than at its position within
`message.option_ciphertexts`. The material listed in
[Reveal Material Persistence] never reaches a relay.

A wallet using relays:

- MUST hand at most one message of a vote to any relay, including on
  retry;
- MUST choose the relay for each message independently and uniformly
  at random from the configuration's `relays`, excluding relays that
  have already received a message of the same vote;
- MUST hand each message over on a separate network connection that
  shares no identifying state with any other hand-off of the same vote,
  for example a fresh Tor circuit or mixnet channel per message (see
  [Network Isolation]);
- MUST NOT hand messages over in share-index order, and SHOULD hand
  them over at independently drawn times rather than in one burst;
- MUST set `submit_at` to the time drawn for that message per
  [Submission Timing].

Where fewer distinct relays remain than messages, the wallet MUST
submit the remaining messages directly via [Direct Share Reveal].

**Response body:**

```json
{"status": "queued"}
```

| Field    | Type   | Description                                                |
| -------- | ------ | ---------------------------------------------------------- |
| `status` | string | `"queued"` if accepted, `"duplicate"` if already received. |
| `error`  | string | Error description (present only on failure).               |

A relay's own obligations (accepting a payload without authenticating
the wallet, requiring no persistent identifier, submitting the message
unaltered) are specified in [^voting-protocol], not here.

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

Querying a vote server (or a relay) for one's own share nullifiers
reveals to it which nullifiers are one's own; this is recorded as an
open issue in [^voting-protocol]. A wallet SHOULD therefore make status
polling optional and off by default. A wallet that does poll SHOULD
issue each query over a fresh network path (see [Network Isolation]),
SHOULD query at most one nullifier per path, and SHOULD do so at times
unrelated to the vote's submission schedule. A wallet MUST NOT require
a confirmed status before proceeding with any other step.

A wallet that observes, by whatever means, that a message has not been
included within a wallet-configured timeout MAY resubmit it under the
retry rules of [^voting-protocol]: directly, or to a relay that has
received no message of the same vote. A duplicate that reaches the
chain is rejected by its nullifier and is harmless.

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

A wallet MUST route each share reveal message — whether submitted
directly or handed to a relay — over a network path that is not shared
with any other message of the same vote — for example, a fresh Tor
circuit or mixnet channel per message. This MUST be the default
behaviour, not an opt-in setting.

A wallet MUST use the same protection for the requests that precede
voting and reveal, in particular commitment tree synchronisation and
PIR queries, and MUST NOT make any request to a vote server, relay, or
PIR endpoint over a path that has carried the wallet's ordinary Zcash
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
window, against `reveal_end_time`, as specified in the "Submission
Timing" section of [^voting-protocol], which adopts the scheduling
discipline ZIP 318 [^zip-0318] defines for pool-crossing transfers. On
the relayed path the drawn times are the `submit_at` values handed to
relays. In summary, and normatively by reference to that document, a
wallet:

- MUST shuffle the vote's shares into a uniformly random order before
  assigning submission times, so that the order in which share values
  are emitted does not depend on their magnitudes or their indices;
- MUST draw each successive inter-submission delay independently from
  an exponential distribution, so that its submissions approximate a
  Poisson process, rather than spacing them evenly or by a fixed
  interval;
- MUST draw all such randomness from a cryptographically secure random
  number generator;
- MUST NOT submit a vote's shares as a single batch.

A wallet MUST NOT place a voter's entire ballot count into a single
share.

**When the window is short.** Where insufficient time remains before
`reveal_end_time`, less the safety margin $\Delta$ that
[^voting-protocol] specifies, to run the full schedule, a wallet MUST
draw each remaining share's submission time independently and
uniformly from the remaining interval, and MUST NOT submit the
remaining shares together.
Submitting promptly is not a substitute for submitting independently: a
wallet that responds to a closing round by sending everything at once
reproduces through timing precisely the exposure that removing
single-share mode was intended to prevent.

Where the remaining window is too short to submit all shares even under
the compressed schedule, a wallet MUST inform the voter before
proceeding rather than submitting silently.

**Returning to reveal.** A vote is counted only if the wallet is opened
at least once during the reveal window, obtains the final VCT root,
constructs its share reveal messages and either submits them or hands
them to relays. A wallet that is not opened between `vote_end_time` and
`reveal_end_time` cannot reveal, and the vote is lost. A wallet SHOULD
surface this to the user when the vote is cast, SHOULD present both
`vote_end_time` and `reveal_end_time`, and SHOULD prompt the user to
return during the reveal window, early enough that the short-window
case above is avoidable, since every option available once the window
is short is worse than having started sooner.

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
domain separator (see [Configuration Authentication]). Version 3 adds
`reveal_end_time`, `relays`, `trustees` and `poll_signature`, and
removes the multi-signer `signatures` array, `ea_pk` and
`min_confirmations` that version 2 carried; its signature is verified
under `ZcashVotingPollSignature:v3`.

## Transaction Lifecycle

### Broadcast Semantics

Transaction submission endpoints (`/delegate-vote`, `/cast-vote`,
`/reveal-share`) return synchronously after initial validation. A
successful response (HTTP 200, `code` = 0) indicates that the
transaction passed validation and entered the mempool. It does not
guarantee inclusion in a block.

### Confirmation Polling

After receiving a successful broadcast response, the wallet SHOULD poll
the [Transaction Status] endpoint using the `tx_hash` from the response.
The transaction is confirmed when the response includes a non-empty
`height` and `code` = 0.

### Timeouts

Transaction validation includes zero-knowledge proof verification,
which may take 30 to 60 seconds. Wallet HTTP clients SHOULD use a
timeout of at least 120 seconds for transaction submission requests.

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
| Client voting library | Determines share decomposition, relay selection and timing behaviour. |

Recording the client library version alone is insufficient: the
circuits determine the meaning of the proofs, and a library version
does not identify them.

Where a wallet implements share decomposition, relay selection or
submission timing itself rather than consuming them from a shared
library, it MUST state this, because such a wallet does not inherit
changes to those behaviours when the library is updated.


# Rationale

## Unified Vote Servers

All chain endpoints — queries and transaction submission, including
direct share reveal — are served under the `/shielded-vote/v1/` path
prefix from the same `vote_servers` base URLs. In the current
architecture, a single `svoted` process hosts every chain endpoint on
the same port.

## Relays Are Listed Separately

The [Relay Hand-off] endpoint is served from the configuration's
`relays` list rather than from `vote_servers`. A wallet must hand each
message of a vote to a distinct relay, so the number of relays a
wallet can use is the number of distinct relay operators the
configuration names; a deployment that hosted the relay endpoint on
the same process as the chain endpoints would give a wallet with one
vote server exactly one relay. Listing relays separately also lets a
deployment keep relay operators disjoint from trustees, which
[^voting-protocol] records as an operational requirement. Nothing
prevents a deployment from operating both a vote server and a relay,
but the wallet treats the two lists independently.

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
- [Snapshot Verification] requires the wallet's Zcash consensus node or
  light client backend to serve the nullifier non-membership tree root
  at a given height. No Zcash node or light client API for this is
  specified; it is a Zcash-node-side interface outside this document,
  and until one exists a wallet cannot complete the verification and
  so cannot take part in a round.


# Reference implementation

A reference implementation of the vote chain REST API, together with
the server-side share submission path that [^voting-protocol] replaces
with relays, is available at
[valargroup/vote-sdk](https://github.com/valargroup/vote-sdk). It does
not yet implement the reveal window, direct share reveal, the relay
hand-off, the poll signature or the acknowledgement query specified
here.

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
