---
title: "Light node API"
parent: Kontrakty
nav_order: 7
---

# Light node API

**Status:** proposed
**Owners:** @IliaPoliak and @Quolen (see [Kontrakty])

The light node is the read-only side of a PoUW-ZK node that light clients
(wallets, the workload generator, [ADR-0010]) talk to. It keeps no mempool
rules of its own and mines nothing. A light client asks it for the chain head,
headers, blocks, account state, the status of a transaction and the status of
a proof job, and it checks the answers against the headers it already holds.

This contract adds no new data formats. Every object is the JSON view from
[node-api.md], and every rule of the [conventions] (typed ids, error shape,
borsh by default, `/v1` prefix) applies unchanged. The light node API is a
subset of the node API plus one view, the proof job, that the full API does not
have as a separate endpoint. Submitting a transaction (`POST /v1/txs`) is not
repeated here: it is the same call as in [node-api.md].

## Terms used below

- A **light node** is a process that serves the endpoints below. In the first
  version it is the full node itself, started with the same binary.
- A **proof job** is a proof request from the moment its transaction is
  accepted until the proof tx is in a block.
- The **head** is the tip of the chain the node follows.

## Port

The light API is served on the node's API port, **8000**, with the full API
(see the compose file of pouw-deploy). A separate port is an open point below.

## Endpoints

All paths have the prefix `/v1`. All are `GET` and read-only.

| Path | Purpose | Answer |
| --- | --- | --- |
| `/v1/light/head` | chain head: tip id and height, chain id | [Head](#head) |
| `/v1/blocks/{height_or_id}/header` | header of one block | `{ "id", "header" }`, see [node-api.md] |
| `/v1/blocks/{height_or_id}` | full block, borsh or JSON | [node-api.md] block view |
| `/v1/headers?from=&to=` | headers in a range, at most 1000 | `{ "headers": [ ... ] }` |
| `/v1/accounts/{addr}` | balance and nonce | [node-api.md] account view |
| `/v1/txs/{txid}` | transaction and its status | [node-api.md] tx view |
| `/v1/txs/{txid}/inclusion-proof` | Merkle inclusion proof | [node-api.md] |
| `/v1/light/proof-jobs/{txid}` | status of a proof job | [Proof job](#proof-job) |

Rules:

- `/v1/light/head` is the cheap poll. A client calls it, compares `height`
  with the last header it holds and fetches the missing headers with
  `/v1/headers`.
- A height beyond the head answers 404 with the code `unknown_block`.
- `/v1/accounts/{addr}` answers 404 for an address the chain has not seen. A
  client treats that as balance 0 and nonce 0 and does not retry.
- The balance and nonce are those of the head. They do not include pending
  transactions. A client that sends several transactions in a row counts the
  nonce itself.
- An answer is valid only for the head it was read at. A client that needs
  consistency between two calls compares `id` of the head before and after.
- Nothing here is trusted by itself. A client checks `parent` links of the
  headers and the inclusion proof against `tx_root` as in
  [the light client flow][flow]. Balance and nonce have no proof in version 1
  (open point).

## Head

```json
{
  "chain_id": 4242,
  "height": 19,
  "id": "blk:02a0a96c92e77c61eba286ea33f748d71bf8dbea317de052491a26dce496ee38"
}
```

## Proof job

`txid` is the txid of the proof request. The status follows the life of the
job:

| `status` | Meaning | Fields present |
| --- | --- | --- |
| `pending` | request accepted, in the mempool | `txid` |
| `included` | a block holds the proof tx | `txid`, `proof_hash`, `height`, `block` |
| `failed` | the node dropped the request (for example the circuit is gone or the request expired) | `txid`, `reason` |

```json
{
  "txid": "tx:61885876c6741433b41af9534b709f4e100743bf2a12d18df406c028f24c0488",
  "status": "included",
  "proof_hash": "proof:3d06c4e368e779ea4b060bb83c1d0eedbcaf251c60d9b66d7a1c426858ead9ba",
  "height": 3,
  "block": "blk:02220a3e49805b849b37d48debc71ef5a75f03aef88c06b7428cf0a031100e4c"
}
```

An unknown txid, or a txid of a transaction that is not a proof request,
answers 404 with the code `unknown_proof_job`. After `included`, the client
asks `/v1/txs/{txid}/inclusion-proof` and verifies it using `proof_hash` as the
leaf, as in [node-api.md].

## Errors

Same shape as everywhere: `{"error": {"code": "<snake_case>", "message": "..."}}`.

| HTTP | `code` | When |
| --- | --- | --- |
| 400 | `bad_request` | an id or a number does not parse |
| 404 | `unknown_block` | height or id not known |
| 404 | `unknown_account` | address never seen |
| 404 | `unknown_tx` | txid never seen |
| 404 | `unknown_proof_job` | not a proof request or never seen |
| 422 | `range_too_large` | `/v1/headers` over 1000 headers |

## Versioning

As in [node-api.md]: adding an endpoint or an optional field keeps `/v1`.
Clients ignore unknown fields.

## Open points

- Separate port for the light API (for example 8001) so it can be exposed
  without the submit endpoint: decides @IliaPoliak.
- Proofs for balance and nonce (a state root in the header) are not part of
  version 1. Until then a light client trusts the node for account state:
  decides @Quolen.
- Whether `failed` jobs are kept, and for how long: decides @IliaPoliak.

## Links

- [node-api.md], the full API and the JSON view.
- [block.md], the block format and the inclusion proof.
- [ADR-0010], the light client.

[Kontrakty]: index.md
[node-api.md]: node-api.md
[block.md]: block.md
[conventions]: node-api.md#conventions
[flow]: node-api.md#a-light-client-verifies-an-inclusion-proof
[ADR-0010]: ../decisions/0010-adr-lahky-klient.md
