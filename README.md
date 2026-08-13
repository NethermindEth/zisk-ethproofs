# Zisk Ethproofs

## EthProofs Client

### Setup

Copy the file `bin/ethproofs-client/example.env` to `.env`, then edit it to set the environment variables:

| Variable               | Description                                                                 |
|------------------------|-----------------------------------------------------------------------------|
| `INPUT_GEN_SERVER_URL` | URL of the input-gen-server                                                 |
| `WEBHOOK_PORT`         | Listening port for the Webhook server                                       |
| `METRICS_PORT`         | Listening port for the Prometheus metrics server                            |
| `COORDINATOR_URL`      | URL of the Zisk Coordinator                                                 |
| `ETHPROOFS_API_URL`    | URL of the EthProofs API                                                    |
| `ETHPROOFS_API_TOKEN`  | Token used to authenticate with the EthProofs API                           |
| `ETHPROOFS_CLUSTER`    | EthProofs cluster ID where proofs will be submitted                         |
| `TELEGRAM_BOT_TOKEN`   | Telegram Bot API token for sending alerts                                   |
| `TELEGRAM_CHAT_ID`     | Telegram chat ID where alerts will be sent                                  |
| `INPUTS_FOLDER`        | Folder where input files will be stored                                     |
| `COMPUTE_CAPACITY`     | Compute capacity required for block proofs                                  |

### Build

To build `ethproofs-client`, run:

```bash
cargo build --release --bin ethproofs-client
```

### Run

To run `ethproofs-client` and start submitting block proofs to EthProofs, run:

```bash
target/release/ethproofs-client -s
```

### Command-line Flags
```
  -s, --submit-ethproofs
          Enable proof submission to Ethproofs
  -t, --telegram-alert <TELEGRAM_ALERT>...
          Send Telegram alerts for specified events [possible values: block-proved, skipped-threshold, proof-failed]
  -d, --insert-db
          Insert block proof data into the database
  -k, --skip-proving
          Skip the proving step (useful for testing)
  -m, --enable-metrics
          Enable Prometheus metrics server
  -i, --keep-input
          Keep the input file after processing
  -b, --skipped-threshold <SKIPPED_THRESHOLD>
          Number of skipped blocks before triggering an alert [default: 5]
  -p, --panic-on-skipped
          Panic when skipped blocks exceed the threshold
```

### Nethermind guest (this fork)

This fork adds an `--input.mode nethermind` that proves blocks with the
[Nethermind ZisK guest](https://github.com/NethermindEth/nethermind/tree/master/src/Nethermind/Nethermind.Stateless.ZiskGuest).
Input generation is delegated to Nethermind's `StatelessInputGen` tool, which emits a fully
framed ZisK input (`<block>.ssz`) from any Nethermind node (>= 1.37) with the `Debug` JSON-RPC
module enabled. The prover-side flow (coordinator, proof kinds, Ethproofs submission) is
unchanged.

Prerequisites:

- A `zisk-coordinator` reachable at `--coordinator-url`. NOTE: stock ZisK v1.0.0-alpha cannot
  prove the Nethermind guest due to two memory-witness bugs (fixes expected in v1.1.0-alpha);
  until then the coordinator must be built from a patched tree
  (see `zisk-mixed-mode.patch` in NethermindEth's ethproofs results, or ask the ZisK team).
- `StatelessInputGen` published as a self-contained binary, same release as the guest ELF
  (input format drifts between releases; tool and guest must be built from the same commit).
- A Nethermind node with `--JsonRpc.EnabledModules` including `Debug`, WS enabled.

Example:

```bash
target/release/ethproofs-client \
  --guest ./elf/nethermind \
  --input.mode nethermind \
  --input.block-modulus 100 \
  --nethermind.inputgen /usr/local/bin/StatelessInputGen \
  --rpc.http-url http://node:8545 \
  --rpc.ws-url ws://node:8545 \
  --prover.executor emulator \
  --ethproofs.submit \
  --ethproofs.api-url https://staging--ethproofs.netlify.app/api/v0 \
  --ethproofs.api-token $ETHPROOFS_API_TOKEN \
  --ethproofs.cluster-id 1
```

Notes:

- `--prover.executor emulator` selects the Rust emulator; the default (`assembly`) matches
  upstream behavior but routes through the ASM planner, which cannot handle the Nethermind
  guest on ZisK v1.0.0-alpha.
- Blocks carrying non-empty EIP-7685 execution requests (~5% of mainnet) are skipped by
  default (opt in to attempting them with `--nethermind.prove-request-blocks`):
  `StatelessInputGen` cannot fetch execution requests over JSON-RPC yet, so those blocks
  would fail stateless validation. The skip happens before the `queued` report, so no
  dangling queued proofs are left on Ethproofs.
- Hints mode (`zisk_hints`) is not supported with the Nethermind guest.

---

## Input Generator Server

### Setup

Copy the file `bin/input-gen-server/example.env` to `.env`, then edit it to set the environment variables:

| Variable             | Description                                                                 |
|----------------------|-----------------------------------------------------------------------------|
| `RPC_URL`            | HTTP JSON-RPC URL for Ethereum Mainnet                                      |
| `RPC_WS_URL`         | WebSocket JSON-RPC URL for Ethereum Mainnet                                 |
| `WS_PORT`            | Listening port for the input-gen-server WebSocket server                    |
| `INPUTS_FOLDER`      | Folder where the block input files are stored                               |
| `BLOCK_MODULUS`      | Modulus used to select which blocks will generate input files               |

### Build

To build `input-gen-server`, run:

```bash
cargo build --release --bin input-gen-server
```

### Run

To run `input-gen-server` and start generating block inputs for the `zec-rsp.elf` client, run:

```bash
target/release/input-gen-server
```

### Command-line Flags
```
  -g, --guest <GUEST>  Guest program for which to generate inputs [default: rsp] [possible values: rsp, zeth]
```
