# Embeddings — Candle (Rust)

The embeddings step is how we turn chunks into vectors using **Candle** (Rust ML)
We intend for embeddings to happen **fully
inside Rust**. 

---

## Purpose

Give the retriever something smart to search over. We embed each text chunk
into a fixed‑length vector so we can run fast similarity search later.

- Input: chunks from [`chunk.md`](./chunk.md)
- Output: vectors + metadata to S3 (see [`state-and-storage.md`](./state-and-storage.md))
- Config: model + dims in [`settings.md`](./settings.md)

---

## Why Candle

**Trust boundary**
- We **trust** Amazon Bedrock **AgentCore** for orchestration (LangGraph control plane).
- We **do not** use other Bedrock services in production for embeddings or chat.

**Why Candle**
- **All‑Rust**: single static binary, no Python, no glibc drama in Lambda.
- **Cost control**: runs on standard AWS Lambda (ARM64) without containers.
- **Performance**: small embedding models + integer/bfloat math work well on Graviton.
- **Portability**: same codepath for local dev and Lambda.
- **Provenance**: Candle is a minimalist Rust ML framework from the Hugging Face team
  (originated by Laurent Mazare & contributors). It focuses on lean CPU/GPU inference.

---

## Model choices

Baseline (good quality / tiny CPU footprint):
- **bge-small-en-v1.5** (384 dims). Solid general embedding model; fast on ARM64.
  - Pros: quality/latency balance, English‑optimized.
  - Cons: smaller vector size can be slightly less precise than base/large variants.

Alternatives (swap in if needed):
- **bge-base-en-v1.5** (768 dims) — higher recall, ~2× compute.
- **e5-small-v2** (384 dims) — similar footprint to bge‑small.
- **all-MiniLM-L6-v2** (384 dims) — widely used, lightweight.

We pin the exact model revision and record it in every output line’s metadata
so indexes are consistent and reproducible.

---

## Files & storage

- **Weights** live in S3 as `safetensors` (plus a small `config.json`).  
- On **Lambda cold start**, we copy them to `/tmp/model/` and keep them hot across
  invokes (Lambda reuses the sandbox).  
- Embedding outputs are written as JSONL to `s3://<bucket>/embeddings/YYYY/MM/DD/*.jsonl`.

Lambda ephemeral storage is configurable; set it high enough to hold model files
and batching buffers (see `settings.md`).

---

## Data flow

1. Read chunk JSONL produced by the chunker:
   ```json
   { "id":"chunk-0001", "text":"Ultron... Vision... Hank Pym...", "meta":{ "entity":"comic", "id":123 } }
   ```
2. Tokenize + encode in **batches** (size tuned in `settings.md`).
3. Produce vector + passthrough metadata:
   ```json
   {
     "id":"chunk-0001",
     "vector":[0.0123, -0.0871, ...],    // len == 384 for bge-small
     "meta":{
       "entity":"comic", "id":123,
       "model_id":"bge-small-en-v1.5@<commit>",
       "pipeline":"candle-embedder@<git-sha>",
       "dims":384
     }
   }
   ```
4. Flush to S3 in rolling JSONL files.

Downstream indexing code assumes consistent `dims`. If you switch models,
create a **new index** (don’t mix dimensions).

---

## Local vs Lambda

**Local (dev)**
- Best for iteration and profiling.
- Same code as Lambda; use a native build and point to a local `./model/` dir.

**Lambda (prod)**
- ARM64 runtime, single Rust binary.
- Copies model from S3 → `/tmp/model` at cold start, then batches requests.
- Memory size controls CPU share. Give it enough to keep p95 latency tight.

---

## Batching & performance knobs

- **batch_size**: start at 16–64; tune based on p95 and memory headroom.
- **max_concurrency**: keep low (1–2 reserved) to avoid stampeding cost.
- **ephemeral storage**: enough for weights + temp buffers (e.g., 1024–4096 MB).
- **warm path**: reuse `/tmp/model` if present; skip S3 download on warm starts.
- **vector dtype**: we emit `f32` for correctness; optional down‑cast to `f16` when
  storing if you accept minor precision loss (document it in metadata).

---

## Code sketch (Rust, pseudocode)

```rust
// crates/embedder/src/main.rs

use candle_core as candle;
use candle_transformers as ct;
use serde_json::Value;
use std::{fs, path::PathBuf};

fn main() -> anyhow::Result<()> {
    // 1) Resolve model dir (local or /tmp/model in Lambda)
    let model_dir = PathBuf::from(std::env::var("MODEL_DIR").unwrap_or("./model".into()));

    // 2) Load tokenizer + model (safetensors)
    let tokenizer = ct::tokenizers::Tokenizer::from_file(model_dir.join("tokenizer.json"))?;
    let device = candle::Device::Cpu;
    let model = ct::models::bert::BertModel::load_from_safetensors(
        &device,
        &model_dir,
        "model.safetensors",
    )?; // exemplar; use the right loader for the chosen embedding model

    // 3) Stream input JSONL → batched encode → write output JSONL
    //    (read chunks, tokenizer.encode_batch, model.forward, pool to fixed-size vector)
    Ok(())
}
```

This is a high‑level outline. The actual crate wires in proper pooling (CLS/mean),
error handling, and JSONL streaming.

---

## Output contract

Every line must include:
- `id` (stable chunk id from chunker)
- `vector` (length = `dims` in `settings.md`)
- `meta` with at least:
  - `model_id`
  - `dims`
  - original passthrough fields needed at query time (entity type, ids, urls, etc.)

See details in [`state-and-storage.md`](./state-and-storage.md) and the retriever’s
filtering rules in [`retrieve.md`](./retrieve.md).

---

## Failure modes

- **Cold start spike**: first request pays S3 → `/tmp` copy. Mitigate with small model and a periodic warmer.
- **Model mismatch**: vectors from different models/dims in one index. Avoid by pinning model and validating `dims` on write.
- **Out of memory**: batch size too large. Drop batch size or raise memory.
- **Slow p95**: increase memory (more CPU), reduce batch, or switch to smaller model.


---

## Pointers

- [`chunk.md`](./chunk.md) — how chunks are made and sized.
- [`retrieve.md`](./retrieve.md) — how vectors are searched.
- [`state-and-storage.md`](./state-and-storage.md) — file layouts, index format.
- [`settings.md`](./settings.md) — knobs for model id, dims, batch size, memory.
- [`orchestrator.md`](./orchestrator.md) — how the control plane calls the embedder.
