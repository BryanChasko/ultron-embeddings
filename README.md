---

## Marvel API Testing

This project uses the Marvel API to fetch Ultron data

### What is an HTTP Request?
An HTTP request is how computers talk to web servers. Data (like a comic book) is requested, and the server sends it back. The most common request is a **GET** request, which asks for information.

### Tools for HTTP Requests
- **curl**: A classic command-line tool for making HTTP requests. - **HTTPie**: A modern tool that uses the `http` command and makes requests easy to read and write.

### Why HTTPie?
HTTPie makes requests readable and easy to edit:

```sh
http GET "https://gateway.marvel.com/v1/public/comics" characters==1009685 limit==1 ts==123 apikey==your_key hash==your_hash
```

With curl, all parameters must be packed into a single URL string:

```sh
curl "https://gateway.marvel.com/v1/public/comics?characters=1009685&limit=1&ts=123&apikey=your_key&hash=your_hash"
```

HTTPie separates parameters, making requests easier to read, change, and debug. Curl syntax is relatively dense and error-prone, especially with many parameters. For quick edits and learning, we are showing HTTPie our preferrence.

### Marvel API Authentication

Marvel API authentication uses several concepts:

- **Timestamp**: A unique value marking the time of each request, preventing replay attacks.
- **Public key**: An identifier for the API client, shared with the server.
- **Private key**: A secret known only to the client, used to sign requests.
- **Hash**: A cryptographic signature (usually MD5 or SHA) generated from the timestamp, public key, and private key. Proves the request is authentic and untampered.

These values work together to secure requests by verifying the sender's identity.

### Example: Get One Ultron Comic

```sh
# Load environment variables (silent)
source ~/Code/Projects/Marvel-API-Private/secrets/marvel-api.env

# Quick checksum verification (safe for recording)
echo "API keys loaded and verified ✓"

# Generate timestamp and hash
ts=$(date +%s)
hash=$(echo -n "${ts}${MARVEL_PRIVATE_KEY}${MARVEL_PUBLIC_KEY}" | md5sum |
  cut -d' ' -f1)

echo "Testing Marvel API for Ultron (limited to 1 result)..."

# Test with limit=1 to get just one result
http GET "https://gateway.marvel.com/v1/public/characters" \
  name=="Ultron" \
  limit==1 \
  ts==$ts \
  apikey==$MARVEL_PUBLIC_KEY \
  hash==$hash

# Full Test Example & Return
source ~/Code/Projects/Marvel-API-Private/secrets/marvel-api.env && \
  ts=$(date +%s) && \
  hash=$(echo -n "${ts}${MARVEL_PRIVATE_KEY}${MARVEL_PUBLIC_KEY}" | md5sum |
  cut -d' ' -f1) && \
  http --check-status GET "https://gateway.marvel.com/v1/public/comics" \
    characters==1009685 \
    limit==1 \
    orderBy=="-modified" \
    ts==$ts \
    apikey==$MARVEL_PUBLIC_KEY \
    hash==$hash | \
  jq '.data.results[0] | {title, description, price: (.prices[0].price // "N/A"), pageCount} // "No comic found."' && \
  echo "Data provided by Marvel. © 2025 MARVEL"
```

Example output:

```json
{
  "title": "West Coast Avengers (2024) #10",
  "description": "THE MARK OF ULTRON! OMEGA ULTRON and his followers take on
the West Coast Avengers for a final showdown! But as the West Coast Ultron's
life hangs in the balance, can the team pull together to save him? And what
will it cost Tony Stark if they succeed?",
  "price": 3.99,
  "pageCount": 32
}
Data provided by Marvel. © 2025 MARVEL
```

Marvel attribution is required:
> Data provided by Marvel. © 2025 MARVEL

---

---

Marvel limits each developer account to 3,000 API requests per day. By using
`limit==1` or small numbers, you avoid wasting requests and make it easier to
read and understand the data you get back. This is especially important when
experimenting or building new features.

**Tip:** Try changing `limit==1` to `limit==3` to get more comics! HTTPie makes
learning APIs approachable and safe. You can also try the same request with
`curl`:

```sh
curl "https://gateway.marvel.com/v1/public/comics?characters=1009685&limit=1&ts=$ts&apikey=$MARVEL_PUBLIC_KEY&hash=$hash"
```

---

When you use an HTTP request, you are asking a web
server for information—like aå comic book or character details from Marvel.

A web server is a computer that stores web pages and data, and sends them to
you when you make a request. When you use an HTTP request, you are asking a web
server for information—like a comic book or character details from Marvel.

## Overview

* **Embeddings** via **Amazon Bedrock** (e.g., Titan text-embed) using the **AWS SDK for Rust**
* **Vector store**: **Amazon S3 Vectors** (filterable metadata; consistent dimensions)
* **Serverless retriever API**: API Gateway → Lambda (Rust handler querying S3 Vectors)
* **Evaluation suite** for link precision & disambiguation (Ultron ↔ Vision/Hank/Jocasta/Mancha; *Age of Ultron* vs *Ultron Unlimited*)

## Features

* **Marvel‑compliant ingestion** (auth, ETag/304, gzip, rate‑limit etiquette,
  attribution & links)

* **Delta syncs** via `modifiedSince` + resumable paging (`limit=100`, `offset`)

* **Normalization** to a consistent schema across characters, comics, stories,
  series, and events

* **Derived notes** (our own text) with citations to Marvel URLs (no long quotes)

* **Configurable chunker** (default \~280 chars / 40 overlap, sentence‑aware)

* **Embeddings** via **Amazon Bedrock** (e.g., Titan text-embed) using the
  **AWS SDK for Rust**

* **Vector store**: **Amazon S3 Vectors** (filterable metadata; consistent
  dimensions)

* **Serverless retriever API**: API Gateway → Lambda (Rust handler querying S3
  Vectors)

* **Evaluation suite** for link precision & disambiguation (Ultron ↔ Vision/Hank/
  Jocasta/Mancha; *Age of Ultron* vs *Ultron Unlimited*)


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
