# State and Storage — S3 + DynamoDB

The state and storage layer holds both the **vector data** used for retrieval and the **checkpoints** that let Lambdas resume cleanly without repeating work. It makes the pipeline durable, queryable, and cheap to scale.

---

## Purpose

- **S3** stores immutable JSONL records: raw ingests, normalized entities, chunks, and embeddings.  
- **DynamoDB** tracks progress and sessions: ingest checkpoints, orchestrator conversations, warm state.  
- Together, they keep the pipeline consistent and resumable.

---

## S3 layout

```
s3://ultron-embeddings-<account-id>/
├── raw/              # untouched Marvel API dumps
│   └── YYYY/MM/DD/*.jsonl
├── derived/          # normalized + derived entities
│   └── YYYY/MM/DD/*.jsonl
├── chunks/           # sentence-aware splits
│   └── YYYY/MM/DD/*.jsonl
├── embeddings/       # Candle vectors
│   └── YYYY/MM/DD/*.jsonl
└── indexes/          # ready-to-query vector shards
    └── model=bge-small-en-v1.5/dims=384/shard-00001.jsonl
```

- Each stage writes append-only files.  
- Indexes are organized by `model_id` and `dims` so retrieval never mixes incompatible vectors.  
- Shards should stay in the 25–200 MB range to balance Lambda cold load vs. scan cost.

---

## DynamoDB state

DynamoDB is used for **lightweight state**:

- **Ingest checkpoints**: track `last_modified` and `etag` per endpoint so Marvel API calls can resume without duplication.  
- **Session memory**: orchestrator conversations can persist here (short-term memory across turns).  
- **Job control**: optional flags for batch jobs (e.g., disable a noisy Lambda, mark a shard as bad).

Example checkpoint table:

| PK                  | SK              | Value                                                                 |
|---------------------|-----------------|-----------------------------------------------------------------------|
| `character#1009685` | `endpoint#comics` | `{ "last_modified":"2025-09-08T00:00:00Z", "etag":"abc123" }` |

---

## Index format

An index shard is just JSONL with vectors + metadata:

```json
{
  "id": "chunk-0001",
  "vector": [0.0123, -0.0871, ...],   // len == dims
  "meta": {
    "entity": "comic",
    "id": 123,
    "urls": ["https://marvel.com/..."],
    "model_id": "bge-small-en-v1.5@<commit>",
    "dims": 384
  }
}
```

- Vectors are stored as arrays of floats (`f32`).  
- Metadata is passthrough from chunker/derive stage plus retrieval-specific fields.  
- Normalization to unit length (L2 norm = 1.0) is done at write time to simplify cosine similarity.

---

## Query flow

1. Retriever Lambda loads a shard from S3 → `/tmp`.  
2. Filters are applied early (`entity`, `id`, `since`, `until`).  
3. Query embedding (via Candle) is compared to shard vectors with cosine similarity.  
4. Top-K results are returned with snippets + metadata.  

---

## Performance & cost knobs

- **Shard size**: bigger shards mean fewer files but longer cold reads; target 25–200 MB.  
- **Reserved concurrency**: limit retriever Lambdas to keep S3 request cost predictable.  
- **Ephemeral storage**: increase if shards or models won’t fit in `/tmp`.  
- **Lifecycle rules**: use S3 lifecycle policies to transition old embeddings to infrequent access or Glacier.  

---

## Pointers

- [`ingest.md`](./ingest.md) — where raw Marvel API data lands.  
- [`embed.md`](./embed.md) — how vectors are produced.  
- [`retrieve.md`](./retrieve.md) — how vectors are queried.  
- [`orchestrator.md`](./orchestrator.md) — how state ties into conversations.  
