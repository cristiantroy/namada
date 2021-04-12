# Anoma ledger prototype

## Quick start

The ledger currently requires that [Tendermint version 0.34.x](https://github.com/tendermint/tendermint) is installed and available on path. [The pre-built binaries and the source for 0.34.8 are here](https://github.com/tendermint/tendermint/releases/tag/v0.34.8), also directly available in some package managers.

There are 2 types of accounts: basic and validator. The accounts have string addresses, basic prefixed with `'b'` and validator with `'v'`. Accounts can have some balance of unspecified currency ¤ (type `u64`).

The transaction code can currently be built from [tx_template](../tx_template) and validity predicates from [vp_template](../vp_template), which is Rust code compiled to wasm.

The transaction template calls `transfer` function from the host environment (Anoma shell) with some hard-coded values for the transfer source, destination and amount (this is temporary until we have more complete storage API for transactions).

The validity predicate template receives the `transfer` data and checks that the transfer's amount > 0.

The validity predicate is currently hard-coded in the shell and used for every account. To experiment with a different validity predicate, build it from the template and restart the shell.

The gossip node needs to toggle the orderbook flag `--orderbook` to relay intents, multiple nodes can be connected with the `--peers` option.

The matchmaker template receives intents with the borsh encoding define in `data_template` and crafts data to be sent with `tx_intent_template` to the ledger. It matches only two intents that are the exact opposite.

```shell
# Install development dependencies
make dev-deps

# Run this first if you don't have Rust wasm target installed:
make -C ../tx_template deps

# Build the validity predicate and transaction wasm from templates, at:
# - ../vp_template/vp.wasm
# - ../tx_template/tx.wasm
make build-wasm-scripts

# Build Anoma
make

# Build and link the executables
make install

# Run Anoma daemon (this will also initialize and run Tendermint node)
make run-anoma

# Reset the state (resets Tendermint too)
cargo run --bin anomad -- reset-ledger

# craft a transaction data to file `tx_data_file`
cargo run --bin anomac -- craft-tx-data --source ba --target va --token xtz --amount 10 --file tx_data_file

# Submit a transaction with a wasm code
cargo run --bin anoma -- tx --path ../tx_template/tx.wasm --data tx_data_file

# Watch and on change run a node (the state will be persisted)
cargo watch -x "run --bin anomad -- run-anoma"

# Watch and on change reset & run a node
cargo watch -x "run --bin anomad -- reset-ledger" -x "run --bin anomad -- run"

# run orderbook daemon with rpc server
cargo run --bin anoma -- run-gossip --rpc --orderbook --matchmaker ../tx_template/tx.wasm --ledger-address  "tcp://127.0.0.1:26658"

# run orderbook daemon with rpc server and matchmaker
cargo run --bin anomad -- run-gossip --rpc --orderbook --matchmaker ../matchmaker_template/matchmaker.wasm --tx-template ../tx_template/tx.wasm --ledger-address "tcp://127.0.0.1:26658"

# craft two opposite intents
cargo run --bin anomac -- craft-intent --address ba --token-buy xtz --amount-buy 10 --token-sell eth --amount-sell 20 --file intent_data_file_A
cargo run --bin anomac -- craft-intent --address va --token-buy eth --amount-buy 20 --token-sell xtz --amount-sell 10 --file intent_data_file_B

# Submit the intents (need a rpc server), hardcoded address
cargo run --bin anomac -- intent --orderbook "http://[::1]:39111" --data intent_data_file_A
cargo run --bin anomac -- intent --orderbook "http://[::1]:39111" --data intent_data_file_B

# Format the code
make fmt
```

## Logging

To change the log level, set `ANOMA_LOG` environment variable to one of:
- `error`
- `warn`
- `info`
- `debug`
- `trace`

To reduce amount of logging from Tendermint ABCI, which has a lot of `debug` logging, use e.g. `ANOMA_LOG=debug,tendermint_abci=warn`.