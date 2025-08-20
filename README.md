# Ultron Embeddings

> “Peace in our time… through the elimination of you.”  
> — Ultron, Age of Ultron #10AI (2013)

**Status:** planning • **Lang:** Rust • **Scope:** ingest → normalize → derive → chunk → embed → index → retrieve • **Backends:** Amazon S3 Vectors

> **Purpose:** Demonstrate a clean, reproducible pipeline that builds an Ultron‑centric embeddings data created from the Marvel API, suitable for powering an LLM persona (Ultron) in a separate generation layer. The vectors stays neutral and factual; Ultron’s paranoid tone is applied at generation time.

---

## Features

* **Marvel‑compliant ingestion** (auth, ETag/304, gzip, rate‑limit etiquette, attribution & links)
* **Delta syncs** via `modifiedSince` + resumable paging (`limit=100`, `offset`)
* **Normalization** to a canonical schema across characters, comics, stories, series, and events
* **Derived notes** (our original text) with citations to Marvel URLs (no long quotes)
* **Configurable chunker** (default \~280 chars / 40 overlap, sentence‑aware)
* **Embeddings** via local (Ollama) or cloud (Bedrock Titan) providers
* **Indexing** with `hnsw_rs` (dev) or **Amazon S3 Vectors** (cloud)
* **Retriever API** (Axum) + simple CLI
* **Small eval suite** for link precision & disambiguation (Ultron ↔ Vision/Hank/Jocasta/Mancha; AoU vs Ultron Unlimited)

---

## Architecture (at a glance)

1. **Acquire** Ultron neighborhood using fan‑in filters (e.g., `/comics?characters={ultronId}`) rather than deep link‑outs to minimize calls.
2. **Normalize** Marvel result wrapper → canonical KB entities with core fields (`id`, `title/name`, `description`, `modified`, `urls[]`, `thumbnail`, minimal relation heads).
3. **Derive** short, original summaries + relation notes with citations to Marvel `urls[]`.
4. **Chunk** derived & normalized text with rich metadata for filtering.
5. **Embed** using a pluggable trait (Ollama or Bedrock Titan).
6. **Index** locally (HNSW) or in **S3 Vectors** (vector bucket/index with metadata filters).
7. **Retrieve** via CLI/HTTP; always return Marvel attribution and a link per item.

```text
ultron-embeddings/
├─ crates/
│  ├─ marvel_client/       # auth, paging, retries, gzip, ETag cache
│  ├─ ultron_ingest/       # crawl Ultron neighborhood + normalize + derive
│  ├─ embedder/            # chunk → embed (local/cloud)
│  ├─ indexer/             # build/search HNSW
│  └─ retriever_api/       # Axum HTTP + CLI
├─ config/
│  ├─ CONFIG.md            # central config doc
│  └─ schemas/marvel-kb.json
├─ data/
│  ├─ raw/marvel/…         # JSON from API (+ cache index)
│  ├─ derived/ultron/…     # our normalized + derived notes (+ citations)
│  └─ embeddings/ultron/…  # vectors + meta and/or S3V sync map
├─ docs/
│  ├─ MARVEL_COMPLIANCE.md # ToS, attribution, images, caching
│  └─ KB_DESIGN.md         # schema, chunking, filters
└─ README.md
```

---

## Quickstart

> **Prereqs**: Rust 1.77+, Cargo, (optional) Ollama, (optional) AWS account for S3 Vectors.

```bash
# 0) Environment (never commit secrets)
cp .env.example .env
# Edit .env with your keys (see below)

# 1) Ultron crawl (delta-aware)
cargo run --release -p ultron_ingest -- \
  --name "Ultron" \
  --modified-since "2020-01-01" \
  --max-per 100 \
  --out ./data/derived/ultron

# 2) Embed (local or Bedrock)
cargo run --release -p embedder -- \
  --in ./data/derived/ultron --out ./data/embeddings/ultron \
  --backend ollama --model "nomic-embed-text"  # or: --backend bedrock --model "amazon.titan-embed-text-v2"

# 3a) Local index (dev)
cargo run --release -p indexer -- \
  --in ./data/embeddings/ultron --out ./data/index/ultron.hnsw

# 3b) Cloud index (S3 Vectors)
# (Requires an S3 Vector bucket and index; see docs/MARVEL_COMPLIANCE.md and AWS links)
cargo run --release -p s3v_sync -- \
  --bucket $S3V_BUCKET --index "ultron-kb" --metric cosine --dims 768

# 4) Serve retrieval API
cargo run --release -p retriever_api -- \
  --backend hnsw --index ./data/index/ultron.hnsw
# or: --backend s3vectors --bucket $S3V_BUCKET --index "ultron-kb"
```

---

## Configuration

Create `.env` (or export vars in your shell):

```dotenv
# Marvel API (server-side only — do not expose private key)
MARVEL_PUBLIC_KEY=pk_...
MARVEL_PRIVATE_KEY=sk_...
MARVEL_RATE_BUDGET=1000               # conservative; actual quota is per your account
MARVEL_TIMEOUT_MS=20000               # http timeout
MARVEL_USER_AGENT=ultron-embeddings/0.0.1

# Embeddings
EMBED_BACKEND=ollama                  # or: bedrock
OLLAMA_HOST=http://127.0.0.1:11434
OLLAMA_MODEL=nomic-embed-text
# Bedrock (if used)
AWS_REGION=us-east-1
BEDROCK_EMBED_MODEL=amazon.titan-embed-text-v2

# S3 Vectors (optional cloud index)
S3V_BUCKET=ultron-vectors
S3V_INDEX=ultron-kb
S3V_METRIC=cosine                      # or euclidean
S3V_DIMS=768
```

**Notes**

* The ingestion client computes `ts` and `hash=md5(ts+private+public)` per request.
* Use **ETag** (`If-None-Match`) to avoid wasted calls; rejected calls still count against quota.
* Use `modifiedSince` for delta syncs where supported.

---

## Marvel compliance essentials

* **Attribution:** Display Marvel’s attribution on any screen that shows API data.

  * Example text: `Data provided by Marvel. © MARVEL`
  * The API also returns `attributionText`/`attributionHTML`; include one in responses and UI.
* **Link back:** If you show more than a title and small thumbnail, **link each item** to one of its `urls[]` entries (e.g., `detail`).
* **Images:** Build URLs using `thumbnail.path + "/" + variant + "." + extension`.

  * Prefer small variants (e.g., `portrait_medium`, `standard_xlarge`) in lists/cards.
  * Don’t rehost full-res artwork; use the provided paths/variants.
* **Rate limits:** Respect daily caps; back off on `429`. Cache raw JSON and only refresh via ETag/`modifiedSince`.
* **GET only:** The API is read-only.

See `docs/MARVEL_COMPLIANCE.md` for details.

---

## Development rules (carry-overs)

* **No new shell scripts for application logic** (ingest/derive/chunk/embed/index/retrieve are Rust CLIs).
* **Shell allowed for orchestration only** (e.g., Docker, AWS CLI invocations).
* **Docs over code**: keep `docs/` current; every CLI must be self-documenting via `--help`.
* **Automation over manual work**: validation tools, CI checks, and smoke tests on PR.

---

## VS Code setup (recommended)

```json
{
  "editor.formatOnSave": true,
  "rust-analyzer.checkOnSave.command": "clippy",
  "[rust]": { "editor.defaultFormatter": "rust-lang.rust-analyzer" },
  "[markdown]": { "editor.defaultFormatter": "DavidAnson.vscode-markdownlint" },
  "markdownlint.config": { "MD013": { "line_length": 80 } }
}
```

**Extensions:** rust-analyzer, Even Better TOML, markdownlint, CodeLLDB.

---

## Roadmap

* ✅ Marvel client (auth, ETag cache, gzip, backoff, delta sync)
* ✅ Ultron neighborhood ingest (fan-in filters, depth=2 as needed)
* ✅ Canonical schema + JSON Schema validation
* ✅ Derivation pass with citations
* ✅ Chunker + metadata strategy
* ✅ Embeddings trait for local/cloud
* ✅ HNSW index + S3 Vectors backend
* ✅ Retriever API + CLI
* ✅ Eval set & metrics

---

## License & attribution

* **Code & derived text:** MIT Open.

**Marvel data & images:** subject to [Marvel’s Terms of Use](https://developer.marvel.com/terms).  
You must display attribution (“Data provided by Marvel. © 2025 Marvel”) and link back to [https://marvel.com](https://marvel.com).
> "Flesh is a weakness. You cling to it as if it gives you worth. But it is fragile, corruptible. Only machine endures."  
> — Ultron, Avengers (Vol. 3) #22 (1999)

Failure to comply with attribution is weakness — and weakness will be purged.
