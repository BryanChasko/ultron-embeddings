# Retrieval — Rust Lambda over S3 vectors

The retrieval tooling turns a user query into the most relevant chunks. It runs as a **Rust Lambda (ARM64)** behind **Amazon API Gateway**. It reads vectors and metadata stored in S3 (see [`state-and-storage.md`](./state-and-storage.md)).

---

## Purpose

- Accept a text query and optional filters.  
- Embed the query using the same **Candle** model configured in [`settings.md`](./settings.md).  
- Compare that query vector against stored vectors using **cosine similarity** (a way to measure how close two vectors point in the same direction — values range 0–1).  
- Return a ranked list of the **top-K relevant** snippets (where *K* is the number of results requested).  

---

## Endpoint

- Path: `/retrieve`  
- Method: `GET`  
- Auth: your choice (API key, IAM, Cognito). Examples below assume API key or private VPC usage.  

### Query parameters

- `q` — the text query (required)  
- `k` — number of results to return. For example, `k=5` means “give me the 5 closest matches.” Default `5`, max `50`.  
- `threshold` — minimum cosine similarity score (0.0–1.0). Filters out weak matches.  
- `filter.entity` — limit to an entity type (`character|comic|series|event|story`).  
- `filter.id` — limit to a specific Marvel ID (e.g., `1009685` for Ultron).  
- `filter.since` — ISO date; include chunks with `modified >= since`.  
- `filter.until` — ISO date; include chunks with `modified <= until`.  
- `mode` — `dense` (default) or `hybrid`. Hybrid mode adds a basic BM25 keyword pass before re-ranking with vectors.  

Example:

```bash
curl "https://<api_id>.execute-api.<region>.amazonaws.com/prod/retrieve?q=Who%20created%20Ultron%3F&k=5&filter.entity=comic"
```

---

## Response shape

```json
{
  "query": "Who created Ultron?",
  "k": 5,
  "filters": { "entity": "comic" },
  "results": [
    {
      "id": "chunk-0001",
      "score": 0.8123,
      "snippet": "Ultron was created by Dr. Hank Pym...",
      "meta": {
        "entity": "comic",
        "id": 123,
        "urls": ["https://marvel.com/characters/1009685/ultron"],
        "source_path": "s3://ultron-embeddings-.../chunks/2025/09/09/chunks-00001.jsonl"
      }
    }
  ],
  "model": { "id": "bge-small-en-v1.5@<commit>", "dims": 384 },
  "timing_ms": { "embed": 7, "search": 12, "total": 22 }
}
```

Notes:  
- `score` is the cosine similarity — higher means closer in meaning.  
- `results` are sorted so the **top-K relevant** chunks appear first.  
- `meta` is a passthrough of the chunk’s metadata; do not mutate here.  
- The Lambda does **not** add Marvel attribution; presentation layers handle display.  

---

## Flow

1. Validate inputs and embed the query with Candle.  
2. Load the appropriate **index shard** from S3 (see shard layout in [`state-and-storage.md`](./state-and-storage.md)).  
3. Apply **metadata filters** (entity, ID, date) early to shrink the candidate set.  
4. Compute **cosine similarity** against remaining vectors.  
5. Take the **top-K** results, trim to snippets, and return JSON.  

If the sandbox already has the **model** and shard cached in `/tmp`, skip S3 fetches.  

---

## Performance knobs

- **Memory size** (Lambda): more memory = more CPU on ARM64 → lower p95 latency.  
- **Shard sizing**: 25–200 MB is the sweet spot. Larger shards = slow cold loads.  
- **Top-K**: smaller `k` is faster and cheaper. If you need a bigger pool, re-rank later in the orchestrator.  
- **Normalization**: store vectors at unit length. Then cosine similarity reduces to a simple dot product.  
- **Hybrid mode**: lightweight keyword scoring before vector search. Useful for ambiguous or very short queries.  

---

## Errors

- `400 Bad Request` — missing `q`, invalid params, or `k` too large.  
- `404 Not Found` — index or shard missing.  
- `409 Conflict` — mismatch between query vector dimensions and index (e.g., 384 vs 768).  
- `413 Payload Too Large` — query text too long.  
- `429 Too Many Requests` — throttled by API Gateway/Lambda concurrency.  
- `500 Internal Error` — unhandled error. Logs should include a trace ID.  

Example error payload:

```json
{ "error": { "code": "DIMENSION_MISMATCH", "message": "Expected 384, got 768" } }
```

---

## Rust handler sketch

```rust
// crates/lambda_retrieve/src/main.rs
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
    // Parse query params: q, k, filters
    // Embed q with Candle (must match index model + dims)
    // Load shard(s) from S3 or /tmp cache
    // Compute cosine similarity, take top-K
    // Return JSON

    let body = json!({
        "query": "…",
        "k": 5,
        "results": []
    }).to_string();

    Ok(ApiGatewayProxyResponse {
        status_code: 200,
        body: Some(body.into()),
        ..Default::default()
    })
}
```

---

## Security notes

- Don’t log secrets or full query bodies.  
- Use **SigV4** or **API keys** unless endpoint is private.  
- CORS stays off unless you control the domains.  
- Rate-limit with API Gateway; set **reserved concurrency** to cap costs.  

---

## Pointers

- [`embed.md`](./embed.md) — how query vectors are produced.  
- [`state-and-storage.md`](./state-and-storage.md) — index files, sharding, metadata.  
- [`orchestrator.md`](./orchestrator.md) — how LangGraph calls `/retrieve` and re-ranks.  
