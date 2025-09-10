# Copilot Instructions — Ultron Data Pipeline

## Project overview

- Rust-based data pipeline that ingests **Marvel API** character data (Ultron-first), produces **JSONL embeddings**, and persists them in **Amazon S3**. Infrastructure is provisioned with **AWS CDK (TypeScript)**.
- Goal: generate clean, working **Rust** code and minimal infra changes that pass CI, follow our style, and respect **Marvel attribution** and licensing requirements.

## Tech stack & layout

- Languages: **Rust ≥ 1.78 (stable)** for app code; **TypeScript** for CDK.
- Storage: **Amazon S3** (`ultron-embeddings-<account-id>`).
- Orchestration: **LangChain inside Lambda** is the **primary orchestrator**; use **Amazon Bedrock AgentCore** only as a **supplementary layer** if additional AWS-native agent integration is required.
- Repo layout:
  - `/src` — Rust crates (binaries + libs)
  - `/infra` — CDK app/stacks
  - `/embeddings` — sample/test JSONL
  - `/.github/workflows` — CI
  - `README.md`, `CONTRIBUTORS.md`

## Build & validate (always run in this order)

1. `cargo fmt -- --check`
2. `cargo clippy -- -D warnings`
3. `cargo build --workspace`
4. `cargo test --workspace` (includes doctests)
5. `cd infra && npm ci && cdk synth` (validate infra)

## Orchestration & agents

- **Primary orchestrator:** `LangChain` running in AWS Lambda (`provided.al2`, `x86_64`).
- **Supplementary orchestration:** Leverage **Amazon Bedrock AgentCore** only if flows require native AWS agentic support beyond LangChain (e.g., chaining Bedrock tools/services).
- Flows should remain **declarative and tool-driven** (fetch → normalize → embed → store). Include retries and explicit error handling.
- For Rust-side orchestration helpers, use **lightweight abstractions** (traits, enums) instead of adding large frameworks.

## Core data workflow (ordered)

1. **Fetch** Marvel character/resource pages (Ultron-first) with authenticated requests and backoff on 429/5xx.
2. **Normalize** responses into a stable schema; filter and clean text.
3. **Embed** normalized text into vectors (project-provided embedding client).
4. **Persist** as **JSONL** (one record per line) and upload to **S3** with tags/metadata.
5. **Index/validate**: schema checks, counts, attribution presence.

## JSONL schema & conventions

- **Schema v1** (one object per line):
  - `id` (string, stable key)
  - `character` (string, e.g., `"Ultron"`)
  - `source` (string, e.g., `"marvel-api"`)
  - `title` (string)
  - `text` (string, cleaned content)
  - `embedding` (array<number>) or `embedding_ref` (string, if stored separately)
  - `url` (string)
  - `ts` (ISO8601 UTC)
  - `attribution` (string; must include Marvel line below)
- File naming: `embeddings/<character>/<YYYYMMDD>/part-####.jsonl`
- Keep JSON lines within practical size limits; chunk long text before embedding.

## S3 write rules

- Bucket: `ultron-embeddings-<account-id>` (provisioned via CDK).
- Set **object tags**: `app=ultron`, `schema=jsonl-emb-1`, `source=marvel-api`.
- Set **user metadata**: `x-amz-meta-sha256`, `x-amz-meta-character`, `x-amz-meta-attribution`.
- Use `Content-Type: application/x-ndjson`. Enforce SSE-S3 or KMS encryption per stack config.

## Marvel attribution (mandatory)

- Any code, docs, data, logs, or outputs containing Marvel content must include:  
  **“Data provided by Marvel. © 2025 MARVEL”**

## Rust coding standards

- Style: `rustfmt` defaults; must pass **Clippy** with `-D warnings`.
- Error handling: return `Result<T, E>`; add context; log once at boundaries.
- Concurrency: use `tokio` async; bound concurrency for network tasks.
- Testing:
  - Unit tests for transforms and schema checks.
  - Doctests for all public APIs.
  - Deterministic fixtures; no live network calls.
- Logging/telemetry: structured logs, include request IDs, attribution aware.
- Public items require doc comments with compiling examples.

## Documentation (Markdown) standards

- Use **ATX headers** (`#`, `##`); exactly one H1 per file.
- Wrap prose at **~80 characters** (except links, tables, or code).
- Prefer **fenced code blocks** with language; avoid indented code blocks.
- Use **reference links** for long URLs; unique, descriptive headings.
- Place `[TOC]` after the intro if supported by host platform.
- Keep docs concise, accurate, and updated alongside code.

## MCP usage (when available)

- In **Agent mode**, use these servers:
  - **GitHub MCP**: search code, open files, branch/PR operations.
  - **Fetch**: retrieve referenced docs when required.
- For multi-file changes: propose a short plan (files, order, purpose) before edits.

## Infra guidance (CDK + Lambda Rust)

- CDK stacks live in `/infra`; must pass `cdk synth` in CI.
- Rust Lambdas:
  - Runtime: `provided.al2`, arch: `x86_64`.
  - Enable X-Ray tracing `Active`.
  - Enforce structured logging, least-privileged IAM roles.
  - Explicit timeouts/memory sized per workload.
  - Load config from Parameter Store/Secrets Manager.

## CI & PR

- CI runs in order: fmt → clippy → build → test → `cdk synth`.
- PRs must include: rationale, files touched, commands run locally, attribution notes.
- Favor small, self-contained patches aligned with workflow above.

## Quick commands

- Build: `cargo build --workspace`
- Test: `cargo test --workspace`
- Lint: `cargo fmt -- --check && cargo clippy -- -D warnings`
- Run example: `cargo run -p ingest -- ./embeddings/ultron.jsonl`
- Infra: `cd infra && npm ci && cdk bootstrap && cdk deploy`
