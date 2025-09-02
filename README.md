# 🦾 Ultron Embeddings — Rust on AWS Lambda

> “Peace in our time… through the elimination of you.”
> — Ultron, *Age of Ultron* #10AI (2013)

**Status:** planning • **Language:** Rust (AWS Lambda) • **Scope:** ingest → normalize → derive → chunk → embed → upsert → retrieve • **Backends:** **Amazon S3 Vectors** (vector store) • **Orchestration:** EventBridge / Step Functions (optional)

> **Purpose:** Build a clean, reproducible pipeline that creates an **Ultron‑focused knowledge base** from the Marvel API and stores it as **vectors in Amazon S3 Vectors**. The **retrieval** layer runs on **AWS Lambda** using **Rust** with the AWS SDKs. The vectors stay neutral and factual; Ultron’s tone is applied only at generation time in a separate layer.

---

## Features

* **Marvel‑compliant ingestion** (auth, ETag/304, gzip, rate‑limit etiquette, attribution & links)
* **Delta syncs** via `modifiedSince` + resumable paging (`limit=100`, `offset`)
* **Normalization** to a consistent schema across characters, comics, stories, series, and events
* **Derived notes** (our own text) with citations to Marvel URLs (no long quotes)
* **Configurable chunker** (default \~280 chars / 40 overlap, sentence‑aware)
* **Embeddings** via **Amazon Bedrock** (e.g., Titan text-embed) using the **AWS SDK for Rust**
* **Vector store**: **Amazon S3 Vectors** (filterable metadata; consistent dimensions)
* **Serverless retriever API**: API Gateway → Lambda (Rust handler querying S3 Vectors)
* **Evaluation suite** for link precision & disambiguation (Ultron ↔ Vision/Hank/Jocasta/Mancha; *Age of Ultron* vs *Ultron Unlimited*)

---

## Architecture (at a glance)

1. **Ingest** Ultron’s “neighborhood” of content using Marvel API fan‑in filters (e.g., `/comics?characters={ultronId}`). Cache raw JSONl in S3 with **ETag** and `modifiedSince`.
2. **Normalize** results → canonical entities with core fields (`id`, `title/name`, `description`, `modified`, `urls[]`, `thumbnail`, relations).
3. **Derive** short summaries + relation notes with citations to Marvel `urls[]`.
4. **Chunk** derived and normalized text with metadata (entity type, ids, arcs).
5. **Embed** chunks using **Bedrock embeddings** (via AWS SDK for Rust).
6. **Upsert** vectors + metadata into **S3 Vectors** (one index for the Ultron knowledge base).
7. **Retrieve** through **API Gateway + Lambda** (Rust retriever over S3 Vectors with metadata filters). Always return Marvel attribution and at least one `urls[]` link.

```text
ultron-embeddings/
├─ crates/
│  ├─ marvel_client/       # auth, paging, retries, gzip, ETag cache (reqwest + serde)
│  ├─ derive/              # normalize -> derive notes (citations)
│  ├─ chunker/             # sentence-aware chunking w/ metadata
│  ├─ embedder/            # Bedrock embeddings via aws-sdk-bedrockruntime
│  ├─ s3v_store/           # S3 Vectors upsert/query (aws-sdk + signed REST calls)
│  └─ lambda_retrieve/     # Lambda handler (API Gateway proxy -> query -> format)
├─ infra/
│  └─ cdk/                 # CDK for S3 Vectors index, buckets, API, Lambdas
├─ docs/
│  ├─ MARVEL_COMPLIANCE.md
│  └─ DESIGN_NOTES.md
└─ README.md
```

---

## Quickstart

> **Requirements**: Rust 1.77+, `cargo-lambda`, AWS CLI configured, access to **Amazon Bedrock** and **S3 Vectors**.

```bash
# 0) Environment (never commit secrets)
cp .env.example .env

# 1) Build workspace
cargo build --release

# 2) Local pipeline (dev): ingest → derive → chunk → embed → upsert
cargo run -p marvel_client -- --character "Ultron" --modified-since 2020-01-01
cargo run -p derive -- --in target/data/raw --out target/data/derived
cargo run -p chunker -- --in target/data/derived --out target/data/chunks
cargo run -p embedder -- --in target/data/chunks --out target/data/embeddings \
  --model-id amazon.titan-embed-text-v2
cargo run -p s3v_store -- upsert --in target/data/embeddings \
  --bucket $S3V_BUCKET --index ultron-kb --metric cosine --dims 1024

# 3) Package & deploy Lambda retriever
cargo lambda build -p lambda_retrieve --release --arm64
cd infra/cdk && npm i && cdk deploy

# 4) Query (dev)
curl "https://<api_id>.execute-api.<region>.amazonaws.com/prod/retrieve?q=Who%20created%20Ultron%3F"
```

---

## Example: retrieve handler

```rust
use aws_lambda_events::apigw::{ApiGatewayProxyRequest, ApiGatewayProxyResponse};
use lambda_runtime::{run, service_fn, Error, LambdaEvent};
use serde_json::json;

#[tokio::main]
async fn main() -> Result<(), Error> {
    run(service_fn(handler)).await
}

async fn handler(event: LambdaEvent<ApiGatewayProxyRequest>)
    -> Result<ApiGatewayProxyResponse, Error>
{
    let q = event.payload.query_string_parameters.first("q").unwrap_or_default();

    // 1) Embed q via Bedrock (Titan v2) → Vec<f32>
    // 2) Query S3 Vectors with cosine similarity + metadata filters
    // 3) Map results → snippets + urls + attribution

    let body = json!({
        "query": q,
        "results": [],
        "attribution": "Data provided by Marvel. © MARVEL"
    }).to_string();

    Ok(ApiGatewayProxyResponse { status_code: 200, body: Some(body.into()), ..Default::default() })
}
```

---

## Configuration

Use environment variables (prefer AWS Secrets Manager/SSM for production):

```dotenv
# Marvel API keys
MARVEL_PUBLIC_KEY=pk_...
MARVEL_PRIVATE_KEY=sk_...
MARVEL_RATE_BUDGET=1000
MARVEL_TIMEOUT_MS=20000
MARVEL_USER_AGENT=ultron-embeddings/0.1.0

# AWS
AWS_REGION=us-east-1
BEDROCK_EMBED_MODEL=amazon.titan-embed-text-v2

# S3 Vectors
S3V_BUCKET=ultron-vectors
S3V_INDEX=ultron-kb
S3V_METRIC=cosine
S3V_DIMS=1024
```

---

## Development rules

* No new shell scripts for app logic (all stages are Rust binaries or Lambda).
* Shell scripts only for orchestration (Docker, CDK/SAM, AWS CLI).
* Documentation over code: keep `docs/` current; every binary should print `--help`.
* Automate validation, CI checks, and smoke tests.

---

## Roadmap

* ✅ Marvel client (auth, ETag cache, gzip, backoff, delta sync)
* ✅ Ultron content ingest (fan‑in filters)
* ✅ Schema + JSON Schema validation
* ✅ Derivation pass with citations
* ✅ Chunker + metadata strategy
* ✅ Bedrock embeddings (Rust SDK)
* ✅ S3 Vectors upsert & filter strategy
* ✅ Retriever Lambda + API Gateway
* ✅ Evaluation set & metrics

---

## License & attribution

* **Code & derived text:** MIT Open.
* **Marvel data & images:** subject to Marvel’s Terms of Use.
  Display attribution (“Data provided by Marvel. © 2025 Marvel”) and link back to [https://marvel.com](https://marvel.com).

> "Flesh is a weakness. You cling to it as if it gives you worth. But it is fragile, corruptible. Only machine endures."
> — Ultron, *Avengers* (Vol. 3) #22 (1999)

Failure to comply with attribution is weakness — and weakness will be purged.
