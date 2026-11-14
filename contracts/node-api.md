---
title: "Node API"
parent: Kontrakty
nav_order: 2
---

# Node API

**Status:** proposed
**Owners:** @IliaPoliak and @Quolen (see [Kontrakty])
**Author of the draft:** @Ddystopia

A PoUW-ZK full node serves this API. Clients submit transactions through it,
the indexer reads the chain through it, light clients read headers and
inclusion proofs, and operators read health and metrics. It is REST. Transactions go in as borsh
bytes or as JSON, and blocks come out as borsh bytes or as a JSON view for
people and tools.
This is not a production system. The goal is to reduce friction between six
repositories in four languages, so the simpler technology wins. gRPC would need
code generation in every language, and protobuf is not a canonical encoding, so
it could not replace borsh for hashing anyway.

The machine-readable form is [node-api.yaml] (OpenAPI 3.1). The
block format is in [block.md] and the event log in
[node-events.md].

**Which block.** This page shows the proof-of-work block the node produces
today. Its header has `parent`, `target`, `nonce`, `chain_id` and `system_id`,
and the id of a block is called `id` (the `block_hash` of [block.md]). The
PoUW header of [block.md] (`difficulty`, buckets, `tx_count`) is not built yet.
When it is, its view takes the place of the header here. One more difference
to [block.md]: `tx_root` of a proof-of-work block is
`sha256(0x02 || u32le(leaf_count) || merkle_root)`, the Merkle root of
[block.md] bound to the number of leaves, so an inclusion proof can carry
the leaf count. The examples come from the fixture chain in
`contracts/fixtures` (chain id 4242).

## Terms used below

- A **txid** is the identifier of a transaction: SHA-256 of the borsh bytes of
  the signed transaction.
- The **mempool** is the set of valid transactions a node has accepted and that
  no block includes yet.
- An **inclusion proof** is the list of sibling hashes that lets anyone
  recompute the Merkle root of a block from one leaf and compare it with the
  root in the header.
- **Long-poll** means the server holds a request open for some time when it has
  nothing to answer, and answers as soon as there is something new.
- A **derived field** is a value that a block does not store and that the node
  computes from the block and the registry (ADR-0007). The API returns it
  computed.
- A **light client** keeps only block headers and asks a full node for
  everything else, then checks the answer against the headers.

## Conventions

These rules apply to the JSON view and to the error bodies.

- **Ids** are strings with a type prefix and 64 lowercase hex digits:
  `tx:<64 hex>`, `blk:<64 hex>`, `circ:<64 hex>`, `addr:<64 hex>`,
  `proof:<64 hex>` (the hash of a proof tx). A block id
  pasted where a txid belongs fails at parse. The OpenAPI file has one named
  schema per type (`TxId`, `BlockId`, `CircuitId`, `Address`, `ProofHash`)
  with the pattern `^tx:[0-9a-f]{64}$`, `^blk:[0-9a-f]{64}$`,
  `^circ:[0-9a-f]{64}$`, `^addr:[0-9a-f]{64}$` and `^proof:[0-9a-f]{64}$`.
- **Other hashes** (`tx_root` and the inclusion proof siblings) are 64
  lowercase hex digits without a prefix.
- **Byte blobs** (`witness`, `proof`, `verifying_key`, `circuit`, `signature`)
  are base64 strings.
- **Field elements** (`public_inputs`, `eta`, `mask`) are 62 lowercase hex
  digits, without a `0x` prefix: the 31 bytes of the FieldInput verbatim, in
  the byte order of borsh (little-endian, like every borsh integer). No byte
  reordering between JSON and borsh.
- **Numbers.** u64 quantities are JSON numbers. The test network never
  exceeds 2^53.
- **Enum tags** are lowercase strings: `"transfer"`, `"proof_request"`,
  `"register_circuit"`, `"groth16"`, `"plonk_kzg"`, `"marlin"`.
- **Errors** have one shape: `{"error": {"code": "<snake_case>", "message": "..."}}`.
  HTTP 400 means malformed, 422 means rejected by the rules (bad signature, bad
  nonce, insufficient balance, unknown circuit, over a limit), 404 means an
  unknown id, 410 means an expired event cursor.
- **Borsh is the default.** Every GET of a block, header or tx answers
  `application/octet-stream` with the borsh bytes. `POST /v1/txs` takes
  `Content-Type: application/octet-stream` with the borsh bytes of a SignedTx,
  or `Content-Type: application/json` with the SignedTx view without `txid`.
  See [JSON view] for the JSON representation.
- **The OpenAPI file is the normative schema.** Every object is closed (no
  extra fields), every string has a pattern and every number a range. In Rust
  each id is a newtype whose parser rejects a string without its own prefix
  (see the types in [block.md]). Other languages are expected to do
  the same.

## Endpoints

All paths have the prefix `/v1` except `/health` and `/metrics`.

| Method | Path | Purpose | Called by |
| --- | --- | --- | --- |
| POST | `/v1/txs` | submit a signed transaction as borsh or JSON, answer 202 with the txid | clients, workload generator, light client |
| GET | `/v1/txs/{txid}` | the tx, its status and, for an included proof request, the derived fields | clients, indexer |
| GET | `/v1/txs/{txid}/inclusion-proof` | Merkle inclusion proof of an included tx | light clients |
| GET | `/v1/blocks/head` | header of the tip with `id` | clients, light clients |
| GET | `/v1/blocks/{height_or_id}` | full block, borsh or JSON view | indexer, explorer |
| GET | `/v1/blocks/{height_or_id}/header` | header of one block | light clients |
| GET | `/v1/blocks?from=&to=` | blocks as a JSON list or length-prefixed borsh, at most 100 per call | indexer |
| GET | `/v1/headers?from=&to=` | headers as a JSON list, at most 1000 per call | light clients |
| GET | `/v1/circuits` | registry entries, without blobs unless `?include=keys` | clients, workload generator |
| GET | `/v1/circuits/{circuit_id}` | one registry entry | clients, miners |
| GET | `/v1/accounts/{addr}` | balance and nonce | clients, light clients |
| GET | `/v1/mempool` | pending txids and counts | clients, dashboards |
| GET | `/v1/events?after=&limit=&wait_ms=` | pull events, long-poll | indexer |
| GET | `/v1/node` | chain id, version, height, peers | operators, clients |
| GET | `/health` | 200 when the node serves | operators, Docker, Jenkins |
| GET | `/metrics` | Prometheus text | Prometheus |

Rules for the endpoints:

- `POST /v1/txs` reads the body as borsh or JSON by `Content-Type`. A body
  that does not decode answers 400. The node decodes JSON into the same
  structure, encodes it to borsh, and checks the signature and computes the
  txid over those bytes. The JSON text itself is never hashed. It checks signature, chain id, nonce, balance, limits and circuit
  existence, then puts the tx in the mempool. The txid in the answer is the
  handle for polling.
- `GET /v1/txs/{txid}` answers 404 for a txid the node has never seen. The
  status is
  `pending` while the tx is in the mempool and `included` once a block holds
  it. For a proof request, the pending tx is the request and the included entry
  is the proof tx.
- `GET /v1/txs/{txid}/inclusion-proof` answers 404 until the tx is included.
  For a proof request the leaf is the proof tx hash. The answer carries
  `leaf_count`, the number of leaves of the tree of the block, because the
  proof-of-work header has no `tx_count`. The `tx_root` of the header commits
  to it.
- `GET /v1/blocks` and `GET /v1/headers` take an inclusive range. A `to` above
  the tip is cut to the tip. A range over the cap answers 422 with the code
  `range_too_large`.
- `GET /v1/headers` answers a JSON list of header views. `GET /v1/blocks`
  answers a JSON list of block views, and with `Accept: application/octet-stream`
  it answers the borsh blocks one after another, each prefixed by its length
  as a u32 (little-endian).
- `GET /v1/accounts/{addr}` answers 404 for an address the chain has not
  seen.
- `/metrics` always contains at least `pouw_node_height`,
  `pouw_node_mempool_size`, `pouw_node_peers` and
  `pouw_node_verification_duration_seconds` (histogram). The full list belongs
  to the metrics contract owned by @Faustynn.

## Flows

### A client submits a proof request and checks inclusion

1. The client reads the circuit and its sender nonce (the tip is at height 2).

   `GET /v1/circuits/circ:08da3fa19442869c4c6bc7ca96d3cc0277acb724a2b1bd01ed173b469e7f091c` answers:

   ```json
   {
     "id": "circ:08da3fa19442869c4c6bc7ca96d3cc0277acb724a2b1bd01ed173b469e7f091c",
     "system": "groth16",
     "constraint_count": 26,
     "owner": "addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c",
     "registered_at": 1
   }
   ```

   `GET /v1/accounts/addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c` answers:

   ```json
   { "address": "addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c", "balance": 999990, "nonce": 1 }
   ```

2. The client builds the body, signs `pouw-zk/tx-sig/v1` plus the borsh bytes
   of the body and sends the body with the signature using `POST /v1/txs` and
   `Content-Type: application/json`:

   ```json
   {
     "body": {
       "chain_id": 4242,
       "sender": "addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c",
       "nonce": 1,
       "fee": 5,
       "kind": "proof_request",
       "circuit": "circ:08da3fa19442869c4c6bc7ca96d3cc0277acb724a2b1bd01ed173b469e7f091c",
       "public_inputs": ["03030303030303030303030303030303030303030303030303030303030303"],
       "witness": "AQIDAw=="
     },
     "signature": "Mj0k23t6GakTYoK1mmKgI1u/kzHH9sF/hKpXrjUEFVlYgNrlf7S82xuKAEOqLoxmxyHvF6ueKzXaUmJpzoYaAA=="
   }
   ```

   The client may send the borsh bytes of the SignedTx instead, with
   `Content-Type: application/octet-stream` (157 bytes or more). The byte
   layout is in [block.md].

   The answer is HTTP 202:

   ```json
   { "txid": "tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488" }
   ```

3. The client polls `GET /v1/txs/tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488` with `Accept: application/json`. While the tx waits:

   ```json
   { "txid": "tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488", "status": "pending", "tx": { "...": "SignedTx view, with txid" } }
   ```

   After a block includes it, the answer carries the derived fields:

   ```json
   {
     "txid": "tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488",
     "status": "included",
     "tx": {
       "txid": "tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488",
       "body": { "...": "as in the ProofRequest view" },
       "signature": "Mj0k23t6GakTYoK1mmKgI1u/kzHH9sF/hKpXrjUEFVlYgNrlf7S82xuKAEOqLoxmxyHvF6ueKzXaUmJpzoYaAA=="
     },
     "height": 3,
     "index": 1,
     "eta": "e9cb0a398184a0c90691827cfb087f938e09811966e3f325b970ea7503cbf4",
     "mask": "1c8cee3341f12fff2e619efdfea4181f6009d23fcfbf8f02807007f1dc9020",
     "proof_system": "groth16",
     "usage_counter": 0,
     "proof": "2+zFHHYfcdCXbrMm5JbKx3GhFw9/YfAobJRg2MBdgHI=",
     "proof_hash": "proof:3d06c4e368e779ea4b060bb83c1d0eedbcaf251c60d9b66d7a1c426858ead9ba"
   }
   ```

4. The client fetches `GET /v1/txs/tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488/inclusion-proof`:

   ```json
   {
     "height": 3,
     "index": 1,
     "leaf_count": 2,
     "siblings": ["cb23d53d0f5ca1e06fcd55ca85d022a5d49c10d426d60577f7a92a610b417c60"]
   }
   ```

5. The client reads `GET /v1/blocks/3/header` and compares the recomputed root
   with `header.tx_root` (see the light-client flow below for the rule).

A rejected submission answers 422, for example:

```json
{ "error": { "code": "bad_nonce", "message": "expected nonce 1, got 0" } }
```

### The indexer reads blocks and follows events

1. The indexer starts with `GET /v1/node` and reads `height`.
2. It reads blocks in ranges of at most 100. `GET /v1/blocks?from=0&to=99`
   with `Accept: application/octet-stream` answers the borsh blocks in
   ascending height order, each prefixed by its u32 length. With `Accept:
   application/json` it answers `{"blocks": [ <Block>, ... ]}` with the shape
   described in [JSON view]. It repeats with
   `from=100` and so on up to the tip.
3. It then follows the event log: `GET /v1/events?after=0&limit=100&wait_ms=20000`.
   The node holds the request up to 20 s when nothing is new. Defaults are
   `after=0`, `limit=100`, `wait_ms=0`. The maxima are 1000 and 30000, and a
   larger value is lowered to the maximum. The first `seq` is 1. The answer has
   `next`, the seq of the last returned event, or `after` when `events` is
   empty:

   ```json
   {
     "events": [
       {
         "seq": 1841,
         "time": "2026-11-02T10:15:30.123Z",
         "node_id": "node-a",
         "kind": "block_accepted",
         "data": {
           "id": "blk:02220a3e49805b849b37d48debc71ef5a75f03aef88c06b7428cf0a031100e4c",
           "height": 3,
           "miner": "addr:66be7e332c7a453332bd9d0a7f7db055f5c5ef1a06ada66d98b39fb6810c473a",
           "tx_count": 1,
           "proof_count": 1
         }
       }
     ],
     "next": 1841
   }
   ```

   The indexer stores the last `seq` it processed and resumes from it after a
   restart. Kinds, their `data` and the 410 answer for an expired cursor are in
   [node-events.md]. On 410 the indexer resyncs blocks through
   `GET /v1/blocks?from=&to=` and continues events from the oldest available
   seq.
4. Blocks are the source of truth. The indexer takes balances and inclusion
   from blocks, and from events only what blocks never contain.

### A light client verifies an inclusion proof

1. It downloads headers: `GET /v1/headers?from=0&to=999` answers
   `{"headers": [ {"id": "blk:...", "header": { ... }}, ... ]}`, at most 1000
   per call. It checks that each `parent` equals the previous `id`.
2. It asks `GET /v1/txs/tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488/inclusion-proof` and takes `index`,
   `leaf_count` and `siblings`. The leaf id is the 32 hash bytes of the txid,
   or of the `proof_hash` for a proof request (the hex after the prefix).
3. It recomputes the Merkle root with the inclusion proof algorithm of
   [block.md][block-inclusion-proof], with `n` taken from `leaf_count` of the
   answer (the proof-of-work header has no `tx_count`), and binds the count:
   `sha256(0x02 || u32le(leaf_count) || root)`. A wrong `leaf_count` does not
   verify, because `tx_root` commits to the true one.
4. The result must equal `tx_root` of the header at `height`. For a proof
   request, the client also checks that the `proof_hash` it got from
   `GET /v1/txs/{txid}` is the leaf id it used.

## For the workload generator

The generator signs with its own key and posts borsh. The node holds no key for
it. The generator sends a `WorkloadTask {task_id, circuit_type, circuit_size,
proof_system, fee, client_id}` ([pouw-workload PR #15],
[pouw-workload#7]). It
becomes a `POST /v1/txs` call as follows:

| WorkloadTask field | In the API |
| --- | --- |
| `circuit_type`, `circuit_size` | select a registered circuit id from the catalog. The generator keeps the table (circuit_type, circuit_size) to `circ:` id, filled when it registers the catalog circuits, and can check it with `GET /v1/circuits` (`constraint_count`) |
| `proof_system` | follows from the circuit. The generator can check it against `system` in the registry entry |
| `fee` | `body.fee` |
| `client_id` | selects the signing key. `body.sender` is its address and `body.nonce` comes from `GET /v1/accounts/{addr}` |
| `task_id` | the txid the node returns in the 202 answer |

The task has no input values. The generator builds `public_inputs` and
`witness` for the chosen circuit type.

Python has the libraries for this: `borsh-construct` for the encoding and
`pynacl` for Ed25519. The test vectors in [block.md][block-test-vectors]
give the exact bytes, the signature and the txid to check an encoder against.

## JSON view

The JSON view is for explorers, dashboards and people. A node accepts borsh
or JSON for a transaction, and both decode to the same bytes. Nothing in the
JSON text is an input to any hash or signature.

A GET of a block, header or tx returns it when the request has
`Accept: application/json`. The view adds the derived fields (ADR-0008). Fields
marked derived are not stored in the block (ADR-0007).

- A tx view has `txid` added.
- A proof tx view adds `proof_hash`, `request_txid` and the derived fields
  `eta`, `mask` and `proof_system`.
- A block view adds `id`.

### Signed transactions

The body is one flat object. `kind` selects which other fields are present
beside the common ones (`chain_id`, `sender`, `nonce`, `fee`). Every view has
`txid`. A request body has the same shape without `txid`.

Transfer:

```json
{
  "txid": "tx:b8000e69f37acaba1e280e18e7c7965aae21b59e2d7bacced559ac226f719bb3",
  "body": {
    "chain_id": 4242,
    "sender": "addr:8139770ea87d175f56a35466c34c7ecccb8d8a91b4ee37a25df60f5b8fc9b394",
    "nonce": 1,
    "fee": 2,
    "kind": "transfer",
    "to": "addr:ed4928c628d1c2c6eae90338905995612959273a5c63f93636c14614ac8737d1",
    "amount": 102
  },
  "signature": "vwqt5cFlDgXYNWnq/afbAO5FLk2dhbcQ6sCbNo+J26E1LdyYB8uQcOTxmUV8g6pAvUhhf7v55rL6W6QTshtlDg=="
}
```

ProofRequest (the body of the first flow, with `txid`):

```json
{
  "txid": "tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488",
  "body": {
    "chain_id": 4242,
    "sender": "addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c",
    "nonce": 1,
    "fee": 5,
    "kind": "proof_request",
    "circuit": "circ:08da3fa19442869c4c6bc7ca96d3cc0277acb724a2b1bd01ed173b469e7f091c",
    "public_inputs": ["03030303030303030303030303030303030303030303030303030303030303"],
    "witness": "AQIDAw=="
  },
  "signature": "Mj0k23t6GakTYoK1mmKgI1u/kzHH9sF/hKpXrjUEFVlYgNrlf7S82xuKAEOqLoxmxyHvF6ueKzXaUmJpzoYaAA=="
}
```

RegisterCircuit:

```json
{
  "txid": "tx:08da3fa19442869c4c6bc7ca96d3cc0277acb724a2b1bd01ed173b469e7f091c",
  "body": {
    "chain_id": 4242,
    "sender": "addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c",
    "nonce": 0,
    "fee": 10,
    "kind": "register_circuit",
    "system": "groth16",
    "constraint_count": 26,
    "verifying_key": "pXOZFKTIjOcwLmNvoDe0IDZAUH4/X9tKx/H9GVtPWFM=",
    "circuit": "Zml4dHVyZSBjaXJjdWl0OiB4ICogeSA9IHo="
  },
  "signature": "U/5XjoqV8nTZFwg+3nYpxr91y1teLiJQQ+Af72AIF5nJz/OfLx65gcZomDANvt2N+HYc8vj6muqMqBlx6MWNAw=="
}
```

The id of the circuit is `circ:` plus the 64 hex digits of this txid.

### Proof tx

An entry of `proof_txs` holds the request, the stored usage counter and the
proof. The other fields are derived:

```json
{
  "proof_hash": "proof:3d06c4e368e779ea4b060bb83c1d0eedbcaf251c60d9b66d7a1c426858ead9ba",
  "request_txid": "tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488",
  "request": {
    "txid": "tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488",
    "body": {
      "chain_id": 4242,
      "sender": "addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c",
      "nonce": 1,
      "fee": 5,
      "kind": "proof_request",
      "circuit": "circ:08da3fa19442869c4c6bc7ca96d3cc0277acb724a2b1bd01ed173b469e7f091c",
      "public_inputs": ["03030303030303030303030303030303030303030303030303030303030303"],
      "witness": "AQIDAw=="
    },
    "signature": "Mj0k23t6GakTYoK1mmKgI1u/kzHH9sF/hKpXrjUEFVlYgNrlf7S82xuKAEOqLoxmxyHvF6ueKzXaUmJpzoYaAA=="
  },
  "usage_counter": 0,
  "proof": "2+zFHHYfcdCXbrMm5JbKx3GhFw9/YfAobJRg2MBdgHI=",
  "eta": "e9cb0a398184a0c90691827cfb087f938e09811966e3f325b970ea7503cbf4",
  "mask": "1c8cee3341f12fff2e619efdfea4181f6009d23fcfbf8f02807007f1dc9020",
  "proof_system": "groth16"
}
```

`eta` and `mask` are the two first public inputs of the circuit, as field
elements. `proof_system` comes from the registry entry of the circuit.
`proof_hash` is the SHA-256 of the borsh bytes of the proof tx.

### Block

```json
{
  "id": "blk:02220a3e49805b849b37d48debc71ef5a75f03aef88c06b7428cf0a031100e4c",
  "header": {
    "version": 1,
    "chain_id": 4242,
    "system_id": 0,
    "height": 3,
    "parent": "blk:0a885d69aaf335cdb81b5fa66053ac6112c66a87b25124cba52ab283d1b37e03",
    "timestamp": 1793600030,
    "miner": "addr:66be7e332c7a453332bd9d0a7f7db055f5c5ef1a06ada66d98b39fb6810c473a",
    "tx_root": "87dca22c438f95bc4316504f0cc38debeff5eeef8d6e0bab50289293c7d0ccd5",
    "target": "0fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
    "nonce": 14
  },
  "txs": [
    {
      "txid": "tx:557929b7ecfe72e03db9d252ea1483ee6ccf13d1798c11630711ea6f1b14c67e",
      "body": {
        "chain_id": 4242,
        "sender": "addr:8139770ea87d175f56a35466c34c7ecccb8d8a91b4ee37a25df60f5b8fc9b394",
        "nonce": 2,
        "fee": 2,
        "kind": "transfer",
        "to": "addr:ed4928c628d1c2c6eae90338905995612959273a5c63f93636c14614ac8737d1",
        "amount": 103
      },
      "signature": "NtsRv1/6fZ5vyDvfBe8ncvQkryw6o42Woa1I2VXyD4zJwij0u0nvs2yVslfVU8TE98/0xFo62+CIwPC+aKFbDw=="
    }
  ],
  "proof_txs": [
    { "proof_hash": "proof:3d06c4e368e779ea4b060bb83c1d0eedbcaf251c60d9b66d7a1c426858ead9ba", "request_txid": "tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488", "...": "as in Proof tx" }
  ]
}
```

`txs` holds Transfer and RegisterCircuit only, never ProofRequest. In the
genesis block `parent` is `blk:` followed by 64 zeros. The header endpoints
answer `{ "id": "blk:...", "header": { ... } }`.

### Circuit entry

Default form:

```json
{
  "id": "circ:08da3fa19442869c4c6bc7ca96d3cc0277acb724a2b1bd01ed173b469e7f091c",
  "system": "groth16",
  "constraint_count": 26,
  "owner": "addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c",
  "registered_at": 1
}
```

With `?include=keys` the entry also has `verifying_key` and `circuit`, both
base64.

### Account

```json
{ "address": "addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c", "balance": 999990, "nonce": 1 }
```

### Mempool and node info

```json
{
  "tx_count": 2,
  "proof_request_count": 1,
  "txids": ["tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488", "tx:557929b7ecfe72e03db9d252ea1483ee6ccf13d1798c11630711ea6f1b14c67e"]
}
```

```json
{ "chain_id": 4242, "version": "0.1.0", "height": 19, "peers": 3 }
```

## Versioning

The path prefix is `/v1`. Adding an endpoint, an optional field or an enum
value keeps the version. Removing or renaming a field or an endpoint, or
changing a meaning, bumps the prefix to `/v2`. Clients ignore unknown fields.

## Open points

- Rate limits per client and per IP: decides @IliaPoliak.
- Is the cap of 100 blocks per `GET /v1/blocks` call enough for the indexer's
  initial sync: decides @NotGeorgeLucas.

## Links

- [block.md], the block format and the hash rules.
- [node-events.md], the event log read through `/v1/events`.
- [ADR-0007], stored and derived fields.
- [ADR-0008], the node returns decoded views and derived fields.
- [pouw-workload#7], task format for the node.
- [pouw-workload PR #15], the current generator.

[Kontrakty]: index.md
[node-api.yaml]: node-api.yaml
[block.md]: block.md
[node-events.md]: node-events.md
[JSON view]: #json-view
[block-inclusion-proof]: block.md#inclusion-proof
[pouw-workload PR #15]: https://github.com/PoUW-ZK/pouw-workload/pull/15
[pouw-workload#7]: https://github.com/PoUW-ZK/pouw-workload/issues/7
[block-test-vectors]: block.md#test-vectors
[ADR-0007]: ../decisions/0007-ulozene-a-odvodene-polia.md
[ADR-0008]: ../decisions/0008-indexer-a-explorer.md
