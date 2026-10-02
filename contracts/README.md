# Sharibo Smart Contracts

This directory contains the Soroban smart contracts for **Sharibo**, private rotating savings circles on Stellar. The payout of the shared pot is anonymized by a real Groth16 zero-knowledge proof, verified on-chain.

## §Entrypoints

| Method | Kind | Auth Requirement | Errors | Purpose |
| --- | --- | --- | --- | --- |
| `create_circle` | write | admin | 9, 10, 11 | Admin creates a circle (Merkle root, contribution, size, vk, fee). |
| `fund` | write | from | 1, 6, 7, 8, 12 | Deposit one `contribution` into the current round's pot. |
| `claim` | write | none (ZK proof) | 1, 2, 3, 4, 5, 8, 11 | Pay the pot to `recipient` given a valid proof. |
| `get_circle` | view | none | 1 | Read circle state. |
| `get_circle_meta` | view | none | 1 | Read mutable/small circle fields — the poll-friendly read. |
| `get_vk` | view | none | 1 | Read the circle's verification key (fetch once, cache it). |
| `get_circle_count` | view | none | - | Count of circles created. |
| `get_round` | view | none | 1 | Get current round for a circle. |
| `get_pot` | view | none | 1 | Get current pot balance. |
| `get_status` | view | none | 1, 7 | Compact status tuple (round, pot, target, cancelled). |
| `get_contributors` | view | none | 1 | Addresses that funded the current round. |
| `has_claimed` | view | none | - | Whether a nullifier has already been used. |
| `propose_admin` | write | admin | 1, 8 | Nominate a new admin. |
| `accept_admin` | write | new_admin (from)| 1, 8 | Accept admin nomination. |
| `expire_round` | write | permissionless-after-deadline | 1, 6, 8, 11, 12 | Refund current-round contributors if deadline passed. |
| `cancel_circle` | write | admin | 1, 8, 11 | Cancel circle and refund current-round contributors. |

`fund(circle_id, from)` requires only `from.require_auth()` — **any address may fund any circle**. The Merkle tree constrains who may _claim_, not who may _fund_.

Before compiling the contracts, ensure you have the proper Rust toolchain installed.

### Prerequisites

- **Rust toolchain**: A Rust channel will eventually be pinned (see `rust-toolchain`).
- **Soroban SDK**: Version 23.
- **WASM target**:
  ```bash
  rustup target add wasm32v1-none
  ```
  *(Note: using `wasm32-unknown-unknown` instead of `wasm32v1-none` is a common trap and will fail; see `docs/troubleshooting.md`)*
- **Stellar CLI**: Ensure you have installed the current `stellar` CLI (superseding the old `soroban` CLI).

### Build Command

To build the contract and compile it to WebAssembly (WASM):

```bash
stellar contract build
```

The compiled WASM artifact will be generated at `target/wasm32v1-none/release/sharibo.wasm`.

### Code Generation, Benchmarks, and Formatting

- **`just xdr-goldens`**: Regenerates the expected XDR layout snapshots (e.g. for the structure-test suite). A contributor whose PR produces a snapshot diff for these should run this and commit the updated snapshots, which is expected when modifying state like the `Circle` struct.
- **`just bench-contract`**: Regenerates CPU instruction benchmarks. Run this and commit the updated snapshots if your changes affect the CPU cost of entrypoints.
- **`cargo fmt` & `cargo clippy`**: Running these locally is highly recommended for style consistency, though neither is strictly enforced in CI currently (see issue #534).

---

## Storage lifetime

Every write entrypoint (`create_circle`, `fund`, `claim`, `cancel_circle`) calls `extend_ttl` on all touched persistent and instance entries. The two constants governing this behaviour are defined and justified in [`contracts/sharibo/src/lib.rs`](sharibo/src/lib.rs):

| Constant | Value | Wall-clock equivalent |
| --- | --- | --- |
| `LEDGER_THRESHOLD` | 100 ledgers | ≈ 8 minutes |
| `LEDGER_EXTEND_TO` | 500,000 ledgers | ≈ 29 days |

The Soroban network maximum for persistent entry TTL is **535,679 ledgers (≈ 30 days)** ([Stellar CLI docs](https://developers.stellar.org/docs/tools/cli/cookbook/extend-contract-wasm)). `LEDGER_EXTEND_TO` is set below that ceiling intentionally, giving a small safety margin while keeping circles live for as long as the network allows.

**What this means in practice**: as long as any participant calls `fund`, `claim`, or `cancel_circle` at least once every 29 days, the circle's storage entry is refreshed and the circle stays accessible indefinitely.

**What to do if a circle goes dormant**: if no write has occurred for longer than the TTL window, the persistent entry will be archived. Before interacting with the circle again, an operator must restore it:

```bash
stellar contract restore \
  --source <admin-or-any-account> \
  --network mainnet \
  --id <contract-id>
```

After a successful `RestoreFootprintOp` the circle's full state (including `round`, `pot`, and `contributors`) is restored with the values it had when it was archived. No data is lost; the circle can then be used normally and the next write will re-extend the TTL to another 29-day window.

`NextCircleId` lives in **instance storage** (`env.storage().instance()`). Soroban instance entries have a TTL measured in ledgers; once a TTL lapses the entry is _archived_ (removed from the live state) and can be restored later via `RestoreFootprintOp`.

**What happens on testnet when instance storage is archived and restored?** After a successful `RestoreFootprintOp` the entry reappears with its last-written value intact — the counter does _not_ reset. The risk is the gap between archival and restoration: any `create_circle` call during that gap would reinitialise the counter to `0` (the `unwrap_or(0)` default), silently overwriting circle 0.
**Storage archival:** every entry the contract writes — instance (`NextCircleId`) and persistent (`Circle`, `Nullifier`) — has its own TTL-extension and archival-consequence analysis, including the `NextCircleId` reset-to-zero risk and the more sensitive nullifier double-claim fence. See [`docs/adr/004-storage-archival.md`](../docs/adr/004-storage-archival.md).

## Schema Version

The `Circle` state uses a versioned layout. The current version is **2** (which introduced `fee_bps` and `fee_recipient`). Since this layout change breaks compatibility with pre-existing persistent state on testnet, deploying it requires a testnet reset (see `docs/runbook-testnet-reset.md`). Future field additions must bump this version number and similarly handle migrations or resets.

---

## Changing the Merkle tree depth

The membership circuit's depth is declared in [`circuits/config.json`](../circuits/config.json)
(`"levels": 4`). The Merkle tree it generates holds `2^levels` commitments at
most, so the contract enforces the same bound at circle creation:

| Constant | Value | Source of truth |
| --- | --- | --- |
| `MAX_CIRCLE_SIZE` (in [`contracts/sharibo/src/lib.rs`](sharibo/src/lib.rs)) | `2^levels = 16` | `circuits/config.json` `levels` |

`create_circle` rejects `size > MAX_CIRCLE_SIZE` with
`Error::InvalidCircleParams`: a larger size would accept funding the tree can
never contain enough members to claim, bricking every round until
`cancel_circle`. A test (`max_circle_size_matches_circuit_levels`) asserts the
constant equals `2^levels` by reading `circuits/config.json`, so a depth change
fails the build loudly.

### Runbook: raising the depth

Because `MAX_CIRCLE_SIZE` is compiled into the contract WASM, a depth change
**requires redeploying the contract** — the bound a deployed instance enforces
cannot change without shipping a new WASM build:

1. Bump `levels` in `circuits/config.json`.
2. Regenerate the circuit (`circuits/scripts/compile.sh`, setup, and proof
   pipeline) so roots/proofs match the new depth.
3. Update `MAX_CIRCLE_SIZE = 2^levels` in `contracts/sharibo/src/lib.rs` — the
   `max_circle_size_matches_circuit_levels` test fails until this matches.
4. Rebuild (`stellar contract build`) and **redeploy**; existing deployments
   keep enforcing the old bound. Any existing circles are unaffected (their
   `size` was validated at creation).

---

## 3. Deploying and Invoking

### Required CLI

Deployments and invocations are performed using the `stellar` CLI.

### Deployment Commands

1. **Deploy the WASM contract onto Testnet**:
   ```bash
   stellar contract deploy \
     --wasm target/wasm32v1-none/release/sharibo.wasm \
     --source admin \
     --network testnet
   ```
   *This command returns the Contract ID (e.g., `CB64IZIBBSPUY63UMIVACKWDKRFNH6WJ2EPAOLM7QR4ZI6IJOT4N2LCF`), which should be recorded in your environment variables.*

2. **Retrieve the Test Token ID (using native XLM Stellar Asset Contract on Testnet)**:
   ```bash
   stellar contract id asset --asset native --network testnet
   ```

### Invocation Example

Creating a circle requires providing exactly **9 arguments** matching the `create_circle` signature (excluding `env` which is injected by the host):

```bash
stellar contract invoke \
  --id <contract-id> \
  --source admin \
  --network testnet \
  -- \
  create_circle \
  --admin <admin-address> \
  --token <token-address> \
  --root <32-byte-hex-root> \
  --contribution 10000000 \
  --size 5 \
  --round_deadline_ledgers 0 \
  --vk '{"alpha":..., "beta":..., "gamma":..., "delta":..., "ic":...}' \
  --fee_bps 0 \
  --fee_recipient <admin-address>
```
*Note: Supplying incorrect arguments (e.g. omitting the new `fee_bps` and `fee_recipient`) results in a Soroban arity error that reads like a toolchain problem, but is simply a parameter mismatch.*

---

## 4. Contract API Reference

Below is the documentation for all public contract methods.

### `create_circle`

* **Signature**:
  ```rust
  pub fn create_circle(
      env: Env,
      admin: Address,
      token: Address,
      root: Fr,
      contribution: i128,
      size: u32,
      round_deadline_ledgers: u32,
      vk: VerificationKey,
      fee_bps: u32,
      fee_recipient: Address,
  ) -> u64
  ```
  (See [`docs/adr/008-protocol-fees.md`](../docs/adr/008-protocol-fees.md) for
  the fee design.)

* **Purpose**:
  Allows an administrator to initialize a new rotating savings circle with a designated payment token, Merkle root containing member commitments, expected contribution amount per member, total circle size (number of members), an optional round deadline (in ledgers), and the Groth16 verification key (`vk`). `fee_bps` (0–10,000 basis points; `0` = no fee) and `fee_recipient` commit an immutable protocol fee paid out of the pot on each `claim`.

* **Preconditions**:
  * The admin must authorize the transaction (`admin.require_auth()`).
  * The contribution amount and circle size must be valid and must not result in an integer overflow when multiplied to determine the pot target.
  * `fee_bps` must be `<= 10_000` (`Error::InvalidFeeParams` otherwise), and when `fee_bps > 0` the `fee_recipient` must not be the contract itself (`Error::InvalidRecipient`).

---

### `fund`

* **Signature**:
  ```rust
  pub fn fund(env: Env, circle_id: u64, from: Address)
  ```

* **Purpose**:
  Deposits exactly one `contribution` amount of tokens into the designated circle's pot for the current round.

* **Preconditions**:
  * The funder must authorize the transfer (`from.require_auth()`).
  * The circle associated with `circle_id` must exist and must **not** be cancelled.
  * The current round's pot must not be full. If the pot has already reached the target (`contribution * size`), further contributions are blocked.
  * The funder must hold a sufficient balance of the circle's configured token.

* **Open Funding Design**:
  Funding is intentionally unshielded and public. Any address can call `fund` on behalf of a circle (not restricted to Merkle root members). This allows external benefactors to top up community pots.

---

### `claim`

* **Signature**:
  ```rust
  pub fn claim(
      env: Env,
      circle_id: u64,
      recipient: Address,
      nullifier_hash: Fr,
      external_nullifier: Fr,
      proof: Proof,
  )
  ```

* **Purpose**:
  Anonymously pays out the round pot (`contribution * size` minus the
  committed protocol fee) to the designated `recipient` address upon
  presenting a valid Groth16 zero-knowledge proof of membership. `claim`
  splits the pot with `apply_fee`: `fee` bps goes to
  `circle.fee_recipient` (the fee transfer is skipped entirely when
  `fee_bps = 0`, keeping the `claim` CPU cost identical to a no-fee
  circle), and the net goes to `recipient`. The `claimed` event reports
  the full pot.

* **Preconditions**:
  * The circle associated with `circle_id` must exist and must **not** be cancelled.
  * The pot must be fully funded (`pot == contribution * size`).
  * The provided `external_nullifier` must match the expected SHA-256 round tag of the current round, computed as `SHA256(circle_id, round) mod r`. This binds the proof to the exact circle and round.
  * The `nullifier_hash` must **not** have been previously used for any claim in this circle.
  * The Groth16 ZK proof must verify successfully against the circle's stored verification key (`vk`) and public inputs (`[nullifier_hash, root, external_nullifier]`).

* **Postconditions**:
  * The nullifier hash is marked as spent in persistent storage.
  * The entire pot balance is transferred to the `recipient` address.
  * The circle's `pot` is reset to `0`, the `round` is incremented by `1`, and the `contributors` list is cleared.

---

### `get_circle`

* **Signature**:
  ```rust
  pub fn get_circle(env: Env, circle_id: u64) -> Circle
  ```

* **Purpose**:
  A view method to retrieve the complete public state and configuration of a circle (e.g., admin, token, Merkle root, round, current pot, and contributors).

* **Preconditions**:
  * The circle associated with `circle_id` must exist.

### `get_circle_meta`

* **Signature**:
  ```rust
  pub fn get_circle_meta(env: Env, circle_id: u64) -> CircleMeta
  ```

* **Purpose**:
  The poll-friendly alternative to `get_circle`: returns the mutable/small
  fields (`schema_version`, `admin`, `token`, `root`, `contribution`, `size`,
  `round`, `pot`, `cancelled`, `round_deadline_ledgers`,
  `round_started_ledger`, `fee_bps`, `fee_recipient`) without the embedded
  `VerificationKey` or the `contributors`/`nullifiers` vectors. On BLS12-381
  the VK alone is several hundred bytes of serialised group elements, so
  callers that poll funding state should prefer this read and fetch the VK
  once via `get_vk`.

* **Preconditions**:
  * The circle associated with `circle_id` must exist.

### `get_vk`

* **Signature**:
  ```rust
  pub fn get_vk(env: Env, circle_id: u64) -> VerificationKey
  ```

* **Purpose**:
  Returns the circle's Groth16 verification key. The VK is committed at
  creation and immutable, so clients fetch it once and cache it (the SDK
  caches per `(contractId, circleId)`).

* **Preconditions**:
  * The circle associated with `circle_id` must exist.

### Events

Every state-changing entrypoint emits a contract event so off-chain observers can react without polling `get_circle`.

| Entrypoint | Topics | Data |
| --- | --- | --- |
| `create_circle` | `("circle", "created", circle_id)` | `(admin, token, contribution, size)` |
| `fund` | `("circle", "funded", circle_id)` | `(from, new_pot, target)` |
| `claim` | `("circle", "claimed", circle_id)` | `(round, amount, recipient)` |
| `cancel_circle` | `("circle", "cancelled", circle_id)` | `(refunded_count, refunded_total)` |

The `claim` event deliberately omits the nullifier hash: publishing it would give observers a linkability handle for correlating anonymized payouts.

---

## 5. Error Code Reference

When a transaction reverts, Soroban returns a typed contract error of the form `Error(Contract, #Code)`.

**For the full, canonical mapping of error codes (1–12) to SDK classes, user-facing messages, and remedies, see [`docs/errors.md`](../docs/errors.md)**.

---

## 6. Test Coverage

The accompanying test suite in [`contracts/sharibo/src/test.rs`](sharibo/src/test.rs) contains **21 tests** verifying the correctness and robustness of the smart contract's state machine, ZK-verification path, and auxiliary mechanisms.

### Key Scenarios Covered

1. **Happy Path Payout**:
   - `happy_path_round_pays_out_and_advances`: Verifies that a fully funded round with a real valid Groth16 proof successfully transfers the pot to a fresh recipient, resets the pot, and increments the round.
2. **Rejection & Error Paths**:
   - `claim_reverts_on_tampered_public_input`: Verifies that `claim` panics with `Error::InvalidProof` when a tampered or invalid nullifier hash is submitted.
   - `claim_reverts_when_underfunded`: Ensures that a claim fails with `Error::RoundNotFunded` if any of the members have not funded.
   - `second_claim_with_same_nullifier_reverts`: Asserts that a nullifier cannot be replayed across rounds, reverting with `Error::AlreadyClaimed`.
   - `claim_reverts_on_stale_round_tag`: Asserts that providing a round tag for a different round reverts with `Error::WrongRoundTag`.
3. **Authorization**:
   - `create_circle_requires_admin_auth` and `fund_requires_member_auth`: Enforces that admin and funder authorizations are properly checked.
4. **Edge Cases**:
   - `sixth_fund_on_full_round_reverts`: Verifies that a sixth deposit on a 5-member circle is blocked with `Error::RoundFull` to prevent over-funding and bricking the claim.
   - `anyone_can_fund`: Verifies the open funding model, ensuring non-member addresses can contribute to a pot.
   - `fund_reverts_on_pot_target_overflow`: Ensures checked multiplication catches overflows.
5. **Cancellations & Refunds**:
   - `cancel_refunds_partial_funders_and_closes_circle`: Verifies that a circle admin can cancel an underfunded circle, automatically refunding all current round contributors in FIFO order and closing the circle permanently.
6. **State Persistence**:
   - `instance_ttl_extended_after_create_fund_claim`: Ensures that `extend_ttl` is executed on all write operations (`create_circle`, `fund`, `claim`) to prevent instance-storage archival issues.
   - `persistent_circle_survives_multiple_rounds_with_ttl_refresh`: Verifies that Circle and Nullifier entries remain accessible across multiple fund/claim rounds when ledger advances by LEDGER_THRESHOLD, confirming TTL is actively re-extended (not once-at-creation).
   - `circle_and_nullifier_entries_individually_extended`: Asserts that both the Circle persistent entry AND the Nullifier persistent entry are independently extended, surviving ledger advancement past LEDGER_THRESHOLD.
   - `ttl_survives_fund_after_ledger_advance`: Confirms that fund operations trigger TTL re-extension even after ledger has advanced past LEDGER_THRESHOLD, allowing indefinite circle activity.
7. **Gas / CPU Benchmarking**:
   - `cpu_instruction_benchmarks`: Benchmarks and prints the precise CPU instructions consumed by write operations (e.g., `create_circle`, `fund`, `claim`) and asserts that they remain safely under the 100M limit.

### TTL (Time-To-Live) & State Archival

The contract uses Soroban's ledger TTL mechanism to manage circle entry lifespan. The following constants govern TTL behavior:

- **`LEDGER_THRESHOLD = 100`**: The minimum ledger distance at which an entry's TTL should be re-extended. At ~5 seconds per ledger, this is ~8.3 minutes. Active circles are re-extended every ~500 seconds of operation.
- **`LEDGER_EXTEND_TO = 500_000`**: The target TTL (in ledgers) after each extension. At ~5 seconds per ledger:
  ```
  500_000 ledgers × 5 sec/ledger = 2_500_000 seconds ≈ 28.9 days ≈ 29 days
  ```
  This gives circles **~1 month of inactivity** before archival risk.

#### Archival & Restoration

If a circle's persistent entry (or any Nullifier) is not written to for 29+ days:

1. **Archival**: The entry moves to the Soroban state archive (temporary inaccessibility). On-chain reads/writes fail with `CircleNotFound`.
2. **Restoration**: The entry can be restored via `RestoreFootprintOp` on the Stellar network (Ledger 50M+ supports historical recovery).
3. **Recovery**: Restoration is **permissionless** — any party can restore an archived circle; no admin key is needed (only network validator consensus).
4. **State Preservation**: Upon restoration, the entry reappears with its last-written value intact (round number, pot, contributors, etc. are preserved).

See the [Soroban Documentation](https://developers.stellar.org/) for "Temporary State" and "State Archival" (Soroban 23.0+).

### Running Coverage (LLVM / Rust)

Contract line coverage is measured with `cargo-llvm-cov` and ratcheted by the
`contracts.lines` entry in [`coverage-thresholds.json`](../coverage-thresholds.json).
That number is a **measured floor** (baseline minus a small margin), not an
aspiration — raise it when coverage improves; never lower it without a
documented reason.

```bash
# Install once (also checked optionally by `just doctor` / scripts/doctor.ts)
cargo install cargo-llvm-cov

# From repo root — enforces the floor and fails if llvm-cov is missing
just coverage

# Or manually from contracts/
THRESHOLD=$(python3 -c 'import json; print(json.load(open("../coverage-thresholds.json"))["contracts"]["lines"])')
mkdir -p coverage
cargo llvm-cov --workspace --tests \
  --ignore-filename-regex='(/tests?/|test\.rs$)' \
  --lcov --output-path coverage/lcov.info
cargo llvm-cov report \
  --ignore-filename-regex='(/tests?/|test\.rs$)' \
  --fail-under-lines "$THRESHOLD"
```

The `--ignore-filename-regex` keeps the floor on production `lib.rs` only
(tests would otherwise inflate the percentage). Measured baseline on
2026-09-28: **74.53%** lines on `lib.rs` → floor **72** in
`coverage-thresholds.json`. See [`COVERAGE_GAPS.md`](COVERAGE_GAPS.md) for
uncovered `panic_with_error!` arms.

CI (`.github/workflows/coverage.yml`) runs the same commands and uploads
`contracts/coverage/lcov.info` as an artifact for reviewers.
