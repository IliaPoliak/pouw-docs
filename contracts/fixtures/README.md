# Fixtures: 20 example blocks

Generated chain of 20 blocks (heights 0 to 19, chain id 4242) in the JSON view of
[node-api.md](../node-api.md). `blocks/block-NN.json` holds one block with its
`id`. `chain.json` holds all blocks and the keys used (seeds are test-only).
`chainspec.json` is the chain spec the chain belongs to: the consensus parameters,
the genesis header and the balances of alice and bob.

**These are proof-of-work blocks**, the format the node produces today: a header
with `parent`, `target`, `nonce` and `system_id`, and mock proofs. Blocks of the
PoUW design in [block.md](../block.md) (`difficulty`, buckets) will look somewhat
different; that format is still being researched, so there are no PoUW fixtures yet.

- Block 1 registers a circuit, most blocks hold a transfer, every third block
  holds a proof request with a mock proof.
- A valid chain of its own spec: every block passes the checks a node runs (proof
  of work, header rules, body rules, the verify of the mock proof and the state
  transition). The blocks are exactly on schedule, 10 seconds apart, so the target
  stays at the easy `initial_target` and a block takes about 16 hashes.
- Built with the node code (`crates/pouw-node/tests/fixtures.rs` in pouw-node)
  through the API views, so ids, Merkle roots, eta, mask, signatures and the derived
  fields are the real ones and the shape is the one the node serves. The id of the
  tip is pinned in that test, so a change of the encoding fails it until the files
  are regenerated.
- The shape names the block id `id`, as the node serves it. `contracts/node-api.yaml`
  still says `hash` and describes the PoUW header; it is changed to match in a
  separate pull request.
- The chain id is 4242, not the 1 of devnet, so a node of devnet does not take these
  blocks for its own. Use the files to mock the API and to test parsers and indexers.

Regenerate:

    POUW_FIXTURES_DIR=<this folder> cargo test -p pouw-node --test fixtures write_fixtures -- --ignored
