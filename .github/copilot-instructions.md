# Copilot Instructions for Ultron Data Project

## Goals
- Collaborate with our developers to write clean, working **Rust code** first and foremost.
- Organize **Marvel API data** (Ultron-focused) into **JSONL embeddings** and manage them in **S3**.
- Always respect **attribution requirements** when handling Marvel data.
- Reduce wasted time: prefer clear, correct completions over exploration.
- Keep generated code aligned with **project structure** and `.md` documentation.

---

## Project Summary
This repository is a **Rust-based data pipeline** for Marvel API character data, starting with **Ultron**.  

It ingests API responses, organizes them into JSONL embeddings, and stores them in **AWS S3** buckets provisioned with CDK.

---

## Core Tools
- **Rust** (>= 1.78 stable) for ingest, transform, and validation.
- **JSONL** as the interchange format for embeddings.
- **AWS S3** for persistent storage (bucket name: `ultron-embeddings-<account-id>`).
- **AWS CDK** (TypeScript) for infrastructure setup under `/infra`.

---

## Build Instructions

### Bootstrap
```bash
# Rust
rustup show        # confirm toolchain
cargo check        # quick compile test

# Infra (Node.js and CDK)
node -v
npm install -g aws-cdk
````

### Build

```bash
cargo build --workspace
```

### Test

```bash
cargo test --workspace
```

### Lint

```bash
cargo fmt -- --check
cargo clippy -- -D warnings
```

### Run

```bash
# Example: run the Ultron ingest binary
cargo run -p ingest -- ./embeddings/ultron.jsonl
```

### Infra

```bash
cd infra
npm install
cdk bootstrap   # only once per account/region
cdk deploy
```

---

## Project Layout

* **README.md** – project overview
* **CONTRIBUTORS.md** – attribution + contributors
* **infra/** – CDK project for S3 bucket
* **embeddings/** – JSONL test and production files
* **src/** – Rust code (binaries + libraries)
* **.github/workflows/** – CI checks

---

## Marvel Attribution

Any code or docs referencing Marvel data **must** state:

> “Data provided by Marvel. © 2025 MARVEL”

* Never output or publish Marvel data without this line.
* Include attribution in logs, examples, and README snippets.

---

## CI Checks

* `cargo build --workspace`
* `cargo test --workspace`
* `cargo fmt -- --check`
* `cargo clippy -- -D warnings`
* `cdk synth` for infra validation

---

## Copilot Behavior

* Assume the user prefers **bias for action**: generate the patch/command, not a question.
* Use **Rust-standard phrasing** for code comments and error messages.
* Use **plainspoken English** for guidance in `.md` files.
* Respect **Marvel attribution rules** at all times.
