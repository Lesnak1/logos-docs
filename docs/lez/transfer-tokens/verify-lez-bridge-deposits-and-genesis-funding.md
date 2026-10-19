---
title: Verify LEZ bridge deposits and genesis funding
doc_type: procedure
product: lez
topics: [lez, bridging, deposits, genesis]
steps_layout: sectioned
authors: moudyellaz, kashepavadan
owner: logos
doc_version: 1
slug: verify-lez-bridge-deposits-and-genesis-funding
sidebar_position: 4
---

# Verify LEZ bridge deposits and genesis funding

#### Run the LEZ test suites that prove a Bedrock deposit credits a zone account exactly once and can't be forged, and fund accounts at genesis on a local zone.

:::tip[Version]
This document is accurate for **Testnet v0.3**.
:::

The bridge is the only way funds enter the [Logos Execution Zone](../../get-started/glossary.md#logos-execution-zone) (LEZ). A user deposits a [note](../../get-started/glossary.md#note) into the zone's [channel](../../get-started/glossary.md#channel) on [Bedrock](../../get-started/glossary.md#bedrock), naming a zone account as the recipient. Once the deposit is final on Bedrock, the zone's sequencer injects a mint into its next block. The bridge [program](../../get-started/glossary.md#program) then moves the amount from the bridge escrow to the recipient's native balance, writes a receipt [PDA](../../get-started/glossary.md#pda) that blocks any replay of the same deposit, and emits a `bridge::Deposit` event.

The bridge escrow starts genesis holding `u128::MAX`, which stands for the whole supply not yet issued. Every balance on the zone descends from a deposit against it, including the balances a zone is created with: the zone has no faucet, so each genesis allocation is a synthetic bridge deposit. User transactions can only increase the escrow, and only the sequencer can mint from it.

There is no first-party command to submit a deposit from Bedrock, and LEZ v0.3.1 disables withdrawals. This procedure therefore checks the mechanism through the tests in the LEZ repository that cover each layer, then shows how to fund accounts at genesis on a local zone. It is intended for zone operators and developers who need to confirm how funds get into a zone and why they can't be forged. To submit deposits from your own sequencer, see [Bridge assets between Logos Blockchain and a Zone](../../blockchain/zone-sdk/bridge-assets-between-blockchain-and-zone.md).

:::info[Prerequisites]

- A supported OS:
   - Linux
   - macOS
- Build dependencies for the LEZ repository:
   - On Ubuntu or Debian:

     ```bash
     sudo apt install build-essential clang libclang-dev libssl-dev pkg-config
     ```

   - On macOS:

     ```bash
     xcode-select --install
     brew install pkg-config openssl
     ```

- [Rust](https://www.rust-lang.org/tools/install), installed with `rustup`. The repository's `rust-toolchain.toml` selects Rust 1.98.1 for you.
- The Risc0 toolchain:

   ```bash
   curl -L https://risczero.com/install | bash
   rzup install
   ```

- [Docker](https://docs.docker.com/get-docker/), running, for the tests in Steps 4 and 5 that start a local Bedrock node.
- `git` and network access to `github.com`.

:::

## What to expect

- You can confirm that the bridge program credits a deposit once and rejects any replay of it.
- You can confirm that a user-submitted deposit never lands, whether public, zero-amount, or private.
- You can fund accounts at genesis on a local zone through the same bridge deposits.

## Step 1: Get the LEZ source

1. Clone the LEZ repository and check out the `v0.3.1` release, which the Testnet v0.3 zone runs:

   ```bash
   git clone https://github.com/logos-blockchain/logos-execution-zone
   cd logos-execution-zone
   git checkout v0.3.1
   ```

1. Check that `rustup` picked up the pinned toolchain:

   ```bash
   rustc --version
   ```

   **Expected result:** `rustc 1.98.1`.

   - If `rustup` reports that the toolchain isn't installed, run `rustup toolchain install 1.98.1`.

Every command in the following steps runs from the repository root. `RISC0_DEV_MODE=1` skips proof generation, which makes the tests much faster without changing what they check.

## Step 2: Check the receipt guard in the bridge program

The bridge program writes a receipt PDA for each deposit, derived from the Bedrock deposit's operation ID. Applying a deposit to a receipt that is already written fails, which is what makes each mint happen exactly once.

1. Run the bridge program's unit tests:

   ```bash
   RISC0_DEV_MODE=1 cargo test --release -p bridge_program
   ```

   **Expected result:** two tests pass:

   ```text
   test tests::a_first_deposit_writes_the_receipt ... ok
   test tests::a_replayed_deposit_cannot_claim_to_be_the_first - should panic ... ok
   ```

   - The second test passes because the replay panics with `Deposit was already processed: its receipt is written`.

## Step 3: Check the deposit event

A deposit transaction emits one `bridge::Deposit` event naming the deposit's operation ID, the recipient, and the amount. Replaying the same transaction fails and emits nothing.

1. Run the deposit event test in the `programs` crate. Its tests only compile with the `artifacts` feature, which embeds the built program binaries:

   ```bash
   RISC0_DEV_MODE=1 cargo test --release -p programs --features artifacts bridge_deposit_emits_one_event_and_its_replay_emits_none
   ```

   **Expected result:** `test inner::tests::bridge_deposit_emits_one_event_and_its_replay_emits_none ... ok`.

## Step 4: Check that forged deposits are dropped

Only the sequencer mints, and only from Bedrock deposits that are final. A witness-less bridge `Deposit` from the sequencer is a fee-exempt system injection. Any other transaction that touches the escrow must be public and may only increase its balance, and the block builder drops every bridge deposit that a user or gossip peer submitted.

1. Run the sequencer's bridge guard tests. They check the settlement guard and the block builder's origin check, and need no Docker:

   ```bash
   RISC0_DEV_MODE=1 cargo test --release -p sequencer_core bridge_guard
   ```

   **Expected result:** two tests pass:

   ```text
   test tests::a_charged_bridge_deposit_is_dropped_by_the_builder_bridge_guard ... ok
   test tests::an_exempt_public_bridge_deposit_is_dropped_by_the_builder_bridge_guard ... ok
   ```

1. Run the bridge integration tests. They start a local zone with a Bedrock node in Docker, submit forged deposits to it, and check that none of them lands:

   ```bash
   RISC0_DEV_MODE=1 cargo test --release -p integration_tests --test bridge
   ```

   **Expected result:** three tests pass:

   ```text
   test private_bridge_deposit_invocation_is_dropped ... ok
   test public_bridge_deposit_invocation_is_dropped ... ok
   test public_bridge_deposit_with_zero_amount_is_rejected ... ok
   ```

   | Test | Forged deposit | What it checks |
   |:-----|:---------------|:---------------|
   | `public_bridge_deposit_invocation_is_dropped` | A public `Deposit` with no witness, sent by a user | The transaction never reaches a block, and the balances of the escrow and the recipient don't change. |
   | `public_bridge_deposit_with_zero_amount_is_rejected` | A public `Deposit` of `0` | A zero-amount deposit leaves the escrow unchanged, and the guard only accepts a strict increase, so it fails too. |
   | `private_bridge_deposit_invocation_is_dropped` | A proven private `Deposit` | Private transactions may not modify the bridge account at all. |

## Step 5: Check the genesis supply

At genesis, the escrow holds `u128::MAX` and nothing else. Each genesis allocation then becomes a deposit against it, with the operation ID `/LEZ/v0.3/GenesisDeposit` followed by the index of the allocation. That ID is a readable string rather than a hash, so no real Bedrock operation can collide with it.

1. Check what the escrow holds at genesis:

   ```bash
   RISC0_DEV_MODE=1 cargo test --release -p testnet_initial_state genesis_system_accounts_have_expected_contents
   ```

   **Expected result:** `test tests::genesis_system_accounts_have_expected_contents ... ok`.

1. Build a zone from its genesis config, without the prebuilt database the other integration tests restore, and check that every configured account holds its configured balance. This test also starts Bedrock in Docker:

   ```bash
   RISC0_DEV_MODE=1 cargo test --release -p test_fixtures --test prebuilt_fixture genesis_from_scratch_follows_config
   ```

   **Expected result:** `test genesis_from_scratch_follows_config ... ok`.

## Step 6: Fund accounts at genesis on a local zone

A local zone has no faucet either. To give accounts a starting balance, list them in the sequencer config's genesis actions; the sequencer turns each entry into a synthetic bridge deposit when it creates the genesis block.

1. Open the sequencer config you run the zone with, for example `lez/sequencer/service/configs/debug/sequencer_config.json`, and add a `supply_account` entry to the `genesis` array for each account to fund. Replace `<account-id>` with the account's base58 ID and `<amount>` with its balance in native units:

   ```json
   "genesis": [
       {
           "supply_account": {
               "account_id": "<account-id>",
               "balance": <amount>
           }
       }
   ]
   ```

   | Field | Purpose |
   |:------|:--------|
   | `account_id` | The public account to credit, in base58. |
   | `balance` | The amount to credit. It is a `u64`, like a Bedrock deposit's amount, and the zone widens it to a `u128` balance. |

   - The `debug` config already funds four accounts this way; edit or extend those entries.

1. Clear the sequencer's stored chain so that the next start builds a new genesis block. The sequencer only applies genesis to an empty database:

   - If you run the zone with `just`, run `just clean`. It removes the sequencer's home, `lez/sequencer/service/sequencer_home`, along with the indexer, wallet, and Docker data of the local zone.
   - If you run `sequencer_service` directly, remove the `rocksdb-<channel-id>` directory and the `bedrock_signing_key` file from the directory the config's `home` points at, where `<channel-id>` is the config's `channel_id`.

1. Start the zone as described in the [LEZ repository's README](https://github.com/logos-blockchain/logos-execution-zone/blob/v0.3.1/README.md#run-the-sequencer-and-node).

   **Expected result:** each account you listed holds its `balance` in its native balance once the genesis block is created.

## Troubleshooting LEZ bridge tests

### A test command reports `running 0 tests`

The test name filter or the feature set left nothing to run, and the command passes without checking anything. Two common causes:

- Tests in the `programs` crate only compile with `--features artifacts`. Without it, `cargo test -p programs` runs no tests.
- The name filter after the package must match part of the test's full path. The bridge program's tests are `tests::a_first_deposit_writes_the_receipt` and `tests::a_replayed_deposit_cannot_claim_to_be_the_first`, so a filter such as `bridge` skips both. Run `cargo test -p bridge_program` without a filter.

Use the commands as given in each step, and check the `test result:` line reports the number of tests the step expects.

### The integration tests fail before any test runs

The tests in Steps 4 and 5 start a Bedrock node in Docker. Check that Docker is running and that your user can start containers, for example with `docker ps`.

### The sequencer panics with `Persisted state does not match the replayed chain`

You changed the genesis actions while the sequencer still had its stored chain from the old config. Remove the stored chain as in [Step 6](#step-6-fund-accounts-at-genesis-on-a-local-zone), or restore the original config.

### A program that calls the bridge fails with `Bridge cannot be invoked through chain calls`

The bridge program only accepts top-level transactions; no other program can call it. Deposits come only from the sequencer, as injected mints or genesis allocations.

### Known constraints

- **No deposit command.** LEZ v0.3.1 has no first-party command to submit a deposit from Bedrock to the zone, and the end-to-end deposit test is disabled while withdrawals are reworked. The happy path is covered by the layered tests in this procedure. A Bedrock deposit carries the recipient in its metadata, and its amount is the whole value of the note it consumes; there is no separate amount field.
- **Withdrawals are disabled.** A `Withdraw` instruction to the bridge program panics with `Withdraws are disabled in the current version of LEZ`.
- **No faucet.** Testnet and local zones get their funds only through bridge deposits, at genesis or from Bedrock.
