# Ingest — Marvel API → Raw JSONL

Data provided by Marvel. © 2025 MARVEL

---

## Purpose

Pull Ultron’s **neighborhood** from the Marvel API and write **immutable raw JSON Lines (JSONL)** to S3.  
This is the *only* stage that talks to Marvel directly; everything downstream replays from S3.

**Neighborhood =** Ultron + directly related entities (characters, comics, stories, events, series) referenced by Marvel endpoints. Keep it tight to stay inside request caps.

---

## Where to run

- **Local dev** — `marvel_client` (Rust binary). Fast iteration, easy debugging, zero AWS cost while you shape schemas and logs.
- **Prod** — Small **Ingest Lambda** on ARM64, scheduled by EventBridge. It only fetches & writes raw; no transforms here.

Why this split: provenance, cheap replays, easier triage. If a later stage changes (chunking, embeddings), you never re-hit Marvel.

---

## Marvel API usage (quick rules)

- **Auth**: send `ts`, `apikey` (public), `hash` = MD5(`ts + private + public`).
- **Rate cap**: 3,000 req/day per key. Test with `limit=1` until the shape is right.
- **Paging**: `limit` ≤ 100, `offset` for full crawls.
- **Delta pulls**: prefer `modifiedSince=YYYY-MM-DD` + `orderBy=-modified`.
- **HTTP cache**: capture `ETag` and `Last-Modified`. Send `If-None-Match` / `If-Modified-Since`. Short‑circuit on **304**.
- **Backoff**: exponential + jitter on **429/5xx**. Respect `Retry-After` if present.
- **Compression**: `Accept-Encoding: gzip` + keep‑alive.
- **User‑Agent**: `ultron-embeddings/0.1.0` (be polite).

Full endpoint and auth notes live in [`docs/marvel-api.md`](./marvel-api.md).

---

## S3 layout (raw only)

```text
s3://ultron-embeddings-<account-id>/
└── raw/
    └── YYYY/
        └── MM/
            └── DD/
                characters-00001.jsonl
                comics-00001.jsonl
                stories-00001.jsonl
                events-00001.jsonl
```

- Append‑only. Never overwrite. Each line is one **Marvel API result object** as returned (no edits).
- Downstream stages read from `raw/` and write their own prefixes (`normalized/`, `derived/`, `chunks/`, `vectors/`).

### S3 object metadata (set at upload)

- `x-amz-meta-source` = `marvel`
- `x-amz-meta-endpoint` = e.g. `characters`, `comics`
- `x-amz-meta-run_ts` = ISO8601 ingest timestamp
- `x-amz-meta-etag` = upstream ETag (if present)
- `x-amz-meta-query` = compact JSON of query args (e.g. `{ "name": "Ultron", "limit": 100, ... }`)

These make estate-wide audits easier. See deeper storage notes in [`docs/state-and-storage.md`](./state-and-storage.md).

---

## JSONL line shape (raw)

One **line** per Marvel `result` object, exactly as received. Example (trimmed for space):

```json
{"id":1009685,"name":"Ultron","description":"…","modified":"2024-07-19T18:30:34-0400","urls":[{"type":"detail","url":"https://marvel.com/characters/1009685/ultron"}],"thumbnail":{"path":"...","extension":"jpg"}}
```

No normalization here. That happens later.

---

## Checkpoints (DynamoDB)

A tiny table avoids re-fetching unchanged pages. Example item:

| PK | SK | Value |
| -----------------------|------------------|----------------------------------------------------------------------- |
| `character#1009685` | `endpoint#comics` | `{ "last_modified": "2025-09-08T00:00:00Z", "etag": "abc123", "offset": 300 }` |

- Update after each successful page write.
- If a run dies mid‑crawl, resume from `offset` and recent `modified` watermark.

---

## Local run (dev)

```sh
# Load secrets (never commit these)
source ~/Code/Projects/Marvel-API-Private/secrets/marvel-api.env

# Ingest Ultron neighborhood since Jan 1, 2020 into a local folder
cargo run -p marvel_client --   --character "Ultron"   --modified-since 2020-01-01   --out ./target/data/raw/   --limit 100 --offset 0 --gzip
```

Outputs: `./target/data/raw/*.jsonl` (same format as S3).

**Good dev loop:** ingest a small slice → inspect lines with `jq` → iterate on normalize/derive/chain logic locally without touching Marvel again.

---

## Lambda run (prod)

- **Trigger**: EventBridge schedule (e.g., nightly at 01:00 UTC).
- **Handler**: Rust Lambda (ARM64) that fetches Marvel pages and streams lines to S3.
- **Timeout/mem**: 30–60s / 512MB is plenty (it’s just HTTP + write).

CDK sketch (trimmed; see `infra/README.md` for full stack):

```ts
new lambda.Function(this, "IngestLambda", {
  runtime: lambda.Runtime.PROVIDED_AL2, // Rust custom runtime
  architecture: lambda.Architecture.ARM_64,
  handler: "bootstrap",
  code: lambda.Code.fromAsset("../target/lambda/marvel_client"),
  environment: {
    MARVEL_PUBLIC_KEY: process.env.MARVEL_PUBLIC_KEY!,
    MARVEL_PRIVATE_KEY: process.env.MARVEL_PRIVATE_KEY!,
    OUTPUT_BUCKET: bucket.bucketName,
    CHECKPOINT_TABLE: table.tableName,
    USER_AGENT: "ultron-embeddings/0.1.0"
  },
  timeout: cdk.Duration.seconds(60),
  memorySize: 512
});
```

Grant S3 `PutObject` on `raw/*` and `PutItem/UpdateItem` on the checkpoint table.

---

## Error handling

- **401/409**: bad hash/params → re-sign and retry once, then fail that page.
- **404**: bad ID or empty collection → log (debug), skip.
- **429**: exponential backoff with jitter; respect `Retry-After`.
- **5xx**: retry with capped attempts; if persistent, advance offset and record a gap marker (so you can audit later).

**Never** log secrets. Keep request/response bodies behind a verbose flag in dev only.

---

## Why not transform here?

Because transforms change. Raw doesn’t. Saving raw lets you:
- re-normalize when the schema evolves,
- re-chunk at a different size/overlap,
- re-embed when you switch models/quantization,
- and debug by diffing stage outputs without re-hitting Marvel or burning your 3,000 req/day.

Transform details live in: [`docs/derive.md`](./derive.md), [`docs/chunk.md`](./chunk.md), [`docs/embed.md`](./embed.md).

---

## Next step

Head to **[`docs/derive.md`](./derive.md)** to turn raw lines into canonical entities with clean fields and stable IDs.