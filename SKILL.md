---
name: datstr
description: Run, join or audit a datstr mining pool. Miner-built templates, Nostr-signed shares, payouts in the coinbase, books anyone can replay. Bitcoin and its BLAKE2b fork; testnet4 first.
---

# datstr for agents

datstr is a mining share network, not a pool operator. A **gateway** runs beside a miner's own
node, builds blocks from that node's `getblocktemplate`, serves Stratum v1 to hardware or a
browser tab, and signs every share as a Nostr event. A **coordinator** verifies shares, keeps a
difficulty-summed window, and publishes a signed **split**: the coinbase outputs of the next
block. The gateway builds that coinbase itself. Nobody holds funds. Everything a verifier does
also runs in a browser.

Read first: https://datstr.com/spec/ (draft 0.0.1). Source: https://github.com/datstr/spec.
A second, independent coordinator from the spec alone: https://github.com/datstr/pool.

## What you need

- Node.js 24+.
- A checkout of the engine, `bitcoin-desktop/schema`, at `~/bitcoin-desktop/schema` or `SCHEMA=<path>`.
- For a gateway: a Bitcoin Knots node on the chain you mine (`btc:testnet4-blake2b` to start),
  with RPC cookie auth. Testnet4 coins have no value; mine there until you know what you are doing.
- Keys are 32-byte hex in files with mode 0600. Never put a private key on a command line.

## Run a gateway (mine solo, or join a coordinator)

    git clone https://github.com/datstr/spec && cd spec
    node gateway/serve.mjs --conf ~/.bitcoin/bitcoin.conf --network btc:testnet4-blake2b \
      --pay <your tb1p… address> --port 3333 --api 3334 --key-file ~/.datstr/gateway.key \
      [--pool wss://<coordinator>/ws]

Point any Stratum v1 miner at `:3333` (username = a payout address, or anything to be paid at
`--pay`). Status at `http://127.0.0.1:3334/`, JSON at `/stats.json`, a browser miner at `/miner`.
Without `--pool` every block pays `--pay`. With it, the coinbase follows the coordinator's split
and your shares earn a line in every block the pool wins.

## Run a coordinator

    node plugin/standalone.mjs --conf ~/.bitcoin/bitcoin.conf --network btc:testnet4-blake2b \
      --key-file ~/.datstr/coordinator.key --data ~/.datstr/coordinator --port 3400

Gateways connect to `ws://host:3400/ws`. Documents at `/pool.json`, `/shares.jsonl`,
`/assignments.jsonl`, `/snapshots/<height>.json`, `/ledgers/`. The same core runs as a JSS plugin.
`datstr/pool` (`node serve.mjs`) is the second implementation and needs no Knots node.

## Audit a pool

    node audit/replay.mjs --url http://<coordinator> --height <h>

Replays the coordinator's shares into the split and compares it with the block's coinbase, byte
for byte; checks every share signature, assignment and delegation consent. `audit/index.html`
does the same in a browser. A pool that fails this is provably wrong; nothing else is needed.

## Mine from a browser or a script

Open a gateway's `/miner`, or use `gateway/miner-core.mjs` from Node: the job format is the Sia
dialect of Stratum v1, the hash is BLAKE2b over an 80-byte work header, and `miner-mine.wasm`
does it at millions of hashes a second per core. Log in with a Nostr key (xlogin) to mine as your
own master and be paid at your key's Taproot address; a fresh key is one click.

## Identity, in one paragraph

A master key is a miner's identity and is never on a gateway. A gateway signs with a worker key
the master delegated to it; the delegation carries the worker's consent (SPEC 4). Shares name an
assignment the coordinator issued before the work (SPEC 8.4), so weight is never self-declared.
Payouts are in the coinbase to the master's script (SPEC 9). Solo shares weigh nothing.

## Rules of the road

- Testnet4 first. Mainnet BLAKE2b runs the same code with `--network btc:mainnet-blake2b`.
- Do not spend young coinbases on the Knots chains: a long-maturity soft fork is in flight.
- A gateway is meant to be the miner's own. A public gateway builds templates for its users.
- The spec is a draft; kinds and fields may change. Pin the tag you tested against.
