# Sharibo — documentation index

Every document in this repository, with a one-line description of what
each covers and where to find it.

---

## Core project docs (root)

| File | Description |
|---|---|
| [`README.md`](../README.md) | Project overview, architecture, on-chain evidence, and fresh-machine setup guide |
| [`NOTES.md`](../NOTES.md) | **Historical** append-only build log — not authoritative for current invariants |
| [`full_product_breakdown.md`](../full_product_breakdown.md) | Complete technical deep-dive: every system layer, engineering decisions, security properties, honest limitations |
| [`CODE_OF_CONDUCT.md`](../CODE_OF_CONDUCT.md) | Contributor code of conduct |
| [`CONTRIBUTING.md`](../CONTRIBUTING.md) | Development workflow, test suites, and dependency audit runbook |
| [`LICENSE`](../LICENSE) | Project license |

## Hackathon-era artifacts (`docs/hackathon/` — point-in-time archive, not maintained)

These files carry an in-file archive notice. Doc-accuracy / structure checks
must skip `docs/hackathon/**` explicitly (see `scripts/doc-archive.mjs`) —
an archive is allowed to contain stale claims.

| File | Description |
|---|---|
| [`hackathon/hackathon_demo_script.md`](hackathon/hackathon_demo_script.md) | Annotated 2m20s demo video script (shot list, voiceover, overlays, recording checklist) |
| [`hackathon/dorahacks_submission.md`](hackathon/dorahacks_submission.md) | DoraHacks submission form text (project description, evidence, honest scope) |
| [`hackathon/VERIFY.md`](hackathon/VERIFY.md) | Historical one-minute judge verification guide (stale testnet IDs; see README for current evidence) |

## Security

| File | Description |
|---|---|
| [`SECURITY.md`](../SECURITY.md) | Security policy and responsible disclosure |
| [`threat-model.md`](threat-model.md) | Assets, adversaries, and which code enforces each property |
| [`poseidon-provenance.md`](poseidon-provenance.md) | Poseidon-over-BLS12-381 constants: packages, verification status, risks |

## Architecture

| File | Description |
|---|---|
| [`architecture.md`](architecture.md) | Detailed version of the README's repository structure: directory ownership, toolchains, and end-to-end data flow |
| [`wire-format.md`](wire-format.md) | Authoritative public signal order and Groth16 byte encodings across circuit, contract, and client |
| [`ceremony.md`](ceremony.md) | **Planned** multi-party trusted-setup runbook (#546) — not executed |
| [`observability.md`](observability.md) | SDK `SdkEvent` taxonomy (`onEvent`) for retries, proofs, artifacts, transactions |

## Architecture decision records (`docs/adr/`)

| File | Description |
|---|---|
| [`adr/001-upgradeability.md`](adr/001-upgradeability.md) | ADR 001: decision to keep the contract immutable and defer admin rotation |
| [`adr/002-multi-round-turn-ordering.md`](adr/002-multi-round-turn-ordering.md) | ADR 002: multi-round turn ordering and cycle-scoped nullifier behavior |
| [`adr/003-client-boundary.md`](adr/003-client-boundary.md) | ADR 003: app ↔ SDK ↔ contract boundary and the current free-function design |
| [`adr/004-storage-archival.md`](adr/004-storage-archival.md) | ADR 004: per-key storage TTL/archival analysis, including the nullifier double-claim fence's residual risk |
| [`adr/005-bls12-381-curve-choice.md`](adr/005-bls12-381-curve-choice.md) | ADR 005: BLS12-381 instead of BN254 (CPU budget) |
| [`adr/006-recipient-binding.md`](adr/006-recipient-binding.md) | ADR 006: `recipientHash` public input for payout binding |
| [`adr/007-leanimt-dynamic-depth-merkle-tree.md`](adr/007-leanimt-dynamic-depth-merkle-tree.md) | ADR 007: LeanIMT dynamic-depth Merkle tree |
| [`adr/008-protocol-fees.md`](adr/008-protocol-fees.md) | ADR 008 (fees): protocol fee on claim |
| [`adr/009-storage-migration.md`](adr/009-storage-migration.md) | ADR 009: schema versioning and storage migration |

## Operations

| File | Description |
|---|---|
| [`canary.md`](canary.md) | Running the e2e suite on a schedule on testnet; foreground-run constraint |
| [`deployment.md`](deployment.md) | How the live browser demo is built and manually deployed to Vercel |
| [`runbook-testnet-reset.md`](runbook-testnet-reset.md) | Runbook for quarterly testnet resets |
| [`troubleshooting.md`](troubleshooting.md) | Common setup and proof-verification failures |

## Reference

| File | Description |
|---|---|
| [`errors.md`](errors.md) | The contract error-code ↔ SDK class table |
| [`glossary.md`](glossary.md) | Plain-language crypto and ROSCA terms |
| [`licenses.md`](licenses.md) | Third-party licence notes |

## Planning

| File | Description |
|---|---|
| [`roadmap.md`](roadmap.md) | Mainnet readiness checklist (no target dates) |

## Audit prep (`docs/audit/` — not an audit report)

| File | Description |
|---|---|
| [`audit/README.md`](audit/README.md) | Audit package index: toolchain pins, repro steps, scope links (#547) |
| [`audit/SCOPE.md`](audit/SCOPE.md) | Draft engagement scope for circuit, setup, contract, client |
| [`audit/NEGATIVE_TESTS.md`](audit/NEGATIVE_TESTS.md) | Existing negative tests and known gaps |

## Circuit docs

| File | Description |
|---|---|
| [`circuits/README.md`](../circuits/README.md) | Circuit build pipeline, BLS12-381 usage, and Poseidon provenance |
| [`circuits/SETUP_TRANSCRIPT.md`](../circuits/SETUP_TRANSCRIPT.md) | Transcript of trusted-setup runs (currently single-contributor) |

## Contract docs

| File | Description |
|---|---|
| [`contracts/README.md`](../contracts/README.md) | Contract build and deploy instructions |
| [`contracts/BENCHMARKS.md`](../contracts/BENCHMARKS.md) | CPU instruction benchmarks and gas analysis for contract entrypoints |

## Configuration examples

| File | Description |
|---|---|
| [`.env.example`](../.env.example) | Example environment variables for the root/scripts |
| [`app/.env.example`](../app/.env.example) | Example environment variables for the browser demo |

---

### Quick links by topic

- **Just getting started:** [`README.md`](../README.md)
- **Historical build narrative:** [`NOTES.md`](../NOTES.md)
- **Wire format / public signals:** [`wire-format.md`](wire-format.md)
- **Deep technical dive:** [`full_product_breakdown.md`](../full_product_breakdown.md)
- **Historical verify checklist (archived):** [`hackathon/VERIFY.md`](hackathon/VERIFY.md)
- **Building the circuit:** [`circuits/README.md`](../circuits/README.md)
- **Building the contract:** [`contracts/README.md`](../contracts/README.md)
- **Architecture decisions:** [`adr/001-upgradeability.md`](adr/001-upgradeability.md), [`adr/002-multi-round-turn-ordering.md`](adr/002-multi-round-turn-ordering.md), [`adr/003-client-boundary.md`](adr/003-client-boundary.md), [`adr/004-storage-archival.md`](adr/004-storage-archival.md), [`adr/005-bls12-381-curve-choice.md`](adr/005-bls12-381-curve-choice.md), [`adr/006-recipient-binding.md`](adr/006-recipient-binding.md), [`adr/007-leanimt-dynamic-depth-merkle-tree.md`](adr/007-leanimt-dynamic-depth-merkle-tree.md), [`adr/008-protocol-fees.md`](adr/008-protocol-fees.md), [`adr/009-storage-migration.md`](adr/009-storage-migration.md)
- **Audit readiness (not audited):** [`audit/README.md`](audit/README.md)

