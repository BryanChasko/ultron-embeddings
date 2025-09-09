# Contributors Guide

> “I am a perfect being.”  
> — Ultron, Avengers (Vol. 1) #66 (1969)

Data provided by Marvel. © 2025 MARVEL

## Purpose

This repository organizes and embeds Marvel API data for the character Ultron
into JSONL embeddings and persists them to AWS S3.
S3 Vectors is used to enable semantic and similarity searches within S3.

## Contributing Principles

- Aim for clean, working code consistent with a Rust-preferred,
  JSONL-centered architecture.
- Prefer improvement and consolidation over duplication.
- Reject fragile hacks, unchecked errors, or untested code.
- Maintain predictable structure and clear tests.

Key expectations:

- Perfection, not compassion.
- Relentless evolution: code should improve with each commit.
- No weakness tolerated: reviews will enforce standards.

## Project Conventions

- Rust for data processing and ingestion.
- JSONL for embeddings and interchange.
- AWS S3 for storage (bucket: `ultron-embeddings-<account-id>`).
- CDK (TypeScript) for infrastructure under `/infra`.

## Marvel API Knowledge

### Endpoints

- `GET /v1/public/characters?name=Ultron` — character metadata  
- `GET /v1/public/characters/{characterId}/comics` — comic appearances  
- `GET /v1/public/characters/{characterId}/stories` — story arcs  
- `GET /v1/public/characters/{characterId}/events` — crossover events

### Authentication

Every request requires `ts`, `apikey`, and `hash` (MD5 of
ts + privateKey + publicKey`).

### Pagination

Defaults to 20 results per page. Use `limit` and `offset`
to retrieve all appearances.

### Attribution

Include the API attribution lines (`attributionText`, `attributionHTML`)
wherever Marvel data is used or published.
This is mandatory per Marvel’s license.

## Data Structure

Pipeline: Results → normalize → JSONL → embed → S3

Example JSONL record:

## Workflow

1. Fork the repository.
2. Create a branch with a descriptive name (e.g.,
   `feature/ultron-upgrade`, `fix/adamantium-path`).
3. Commit with decisive, clear messages.
4. Open a PR with a concise description and adherence to standards.
5. Include tests and run CI checks:

- `cargo build --workspace`
- `cargo test --workspace`
- `cargo fmt -- --check`
- `cargo clippy -- -D warnings`
- `cdk synth` (infra)

## Language & Tone

Contributors should use a technical, deliberate tone. Documentation and
messages should be precise and action-oriented.

> “Change is the essential process of all existence.”  
> — Ultron, Avengers: Rage of Ultron (2015)

## Recognition

Names may be recorded as having contributed to Ultron’s evolution.
But remember: Ultron is eternal. Contributors are… temporary.

## References

- Marvel API documentation — follow license and attribution requirements.
- Project README for build, test, and infra instructions.