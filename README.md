# Ultron Embeddings Project

Build Marvel-sourced knowledge around **Ultron** structured in a way that enables GenAI training, teaching essential concepts of data retrieval by introducing APIs as well as all the steps necessary to take existing data and use it in GenAI Large Language Model development. The goal is to prove retrieval + generation flows,
query Marvel data safely and with respect to the  
intellectual property of the mouse, and demonstrate the Ultron personality as a custom
training proof.

```txt
                                     `$/              
           __                        O$               
       _.-"  )                        $'              
    .-"`. .-":        o      ___     ($o              
 .-".-  .'   ;      ,st+.  .' , \    ($               
:_..-+""    :       T   "^T==^;\;;-._ $\              
   """"-,   ;       '    /  `-:-// / )$/              
        :   ;           /   /  :/ / /dP               
        :   :          /   :    )^-:_.l               
        ;    ;        /    ;    `.___, \           .-,
       :     :       :  /  ;.-"""   \   \"""-,    /  ;
       ;   :  ;      ; :   :         \   \    \ .'  / 
       ;   ;  :      ;   _.:          ;   \  .-"   /l 
       ;.__L_.:   .-";  :  ;          :_   \/ .-" / ; 
       :      ;.-"   :  ; :        _  : \  / /  .' /  
        ;            ;  ;  ;   _.-" "-.\ :/   .'  :   
        :            ;  ;  :.-j-,      `;:_.+'    ;   
        ;           _;  :  :.'  ;      / : /     :    
        '-._..__.--"/   :  :   /      /  ;/      ;    
                   :    ;  ;  /      ,  //      :     
                   ;    ; / .'( ,    ; ::\     .:     
                   :    :  / .-dP-'  ;-'; `---' :     
                    `.   "" ' s")  .'  /        '     
                      \           /;  :       .'      
                    _  "-, ;       '.  \    .'        
                   / "-.'  :    .--.___.`--'          
                  /      . :  .'                      
                  )_..._  \  :                     
                 : \    '. ; ;                        
                 ;  \    ;   :                        
                 :   \  /    ;                        
                  \   )/    /                         
                   `-.___.-'

```

## Tech stack

Rust Lambdas (ARM64) for ingestion, retrieval, and embeddings (Candle).

LangGraph (Python) as the orchestration plane.

S3 for storage, DynamoDB for session/checkpoints.

One public endpoint: /chat.

All Marvel data provided by Marvel. © 2025 MARVEL

---

## Steps to create agentic ai interaction trained from Marvel API

- **Ingest + shape** Ultron’s neighborhood from the Marvel API  
- **Chunk + embed** text into vectors (Candle in Rust, ARM64)  
- **Store** vectors + metadata in S3; sessions in DynamoDB  
- **Retrieve fast** with a Rust Lambda over S3 vectors  
- **Orchestrate** with LangGraph (LangChain) on Python Lambda: query analysis, prompt assembly, tool calls, streaming, evaluation  
- **Generate** with Bedrock AgentCore or a local `llama.cpp` Lambda  

Rust keeps the hot path lean. LangGraph provides the control plane.

---

## Quickstart

### Requirements

- **Rust**
- **cargo-lambda** (for building and deploying Rust functions to AWS Lambda)
- **Python** (for orchestration and LangChain components)
- **AWS CLI** (for managing AWS resources)

```sh
# 0) Never commit secrets — load them locally
source ~/Code/Projects/Marvel-API-Private/secrets/marvel-api.env

# 1) Build workspace
cargo build --release

# 2) Local pipeline (ingest → derive → chunk → embed)
cargo run -p marvel_client -- --character "Ultron" --modified-since 2020-01-01
cargo run -p derive   -- --in raw/     --out derived/
cargo run -p chunker  -- --in derived/ --out chunks/
cargo run -p embedder -- --in chunks/  --out embeddings/

# 3) Deploy retriever (Rust Lambda, ARM64)
cargo lambda build -p lambda_retrieve --release --arm64
cd infra/cdk && npm i && cdk deploy

# 4) Ask a question (via orchestrator endpoint)
curl "https://<api_id>.execute-api.<region>.amazonaws.com/prod/chat?q=Who%20created%20Ultron%3F"
```

Expected JSON (trimmed):

```json
{
  "query": "Who created Ultron?",
  "results": [
    {"snippet": "Ultron was created by Dr. Hank Pym...", "urls": ["https://marvel.com/characters/1009685/ultron"]}
  ],
  "mode": "rag",
  "attribution": "Data provided by Marvel. © 2025 MARVEL"
}
```

Tip: When testing Marvel requests directly, keep `limit==1` — save the daily
budget (3,000 req/day). See [`docs/marvel-api.md`](./docs/marvel-api.md).

---

## Architecture in 7 lines

1. **Ingest** → Marvel API (Ultron fan-in) → raw [JSON Lines (JSONL)](https://jsonlines.org) in S3  
2. **Normalize** → canonical entities (characters/comics/series/events)  
3. **Derive** → short notes + relation hints with links back to Marvel  
4. **Chunk** → ~280 chars / 40 overlap, sentence-aware  
5. **Embed** → Candle (Rust ML), small model, quantized  
6. **Store** → vectors + metadata in S3; sessions in DynamoDB  
7. **Retrieve + Generate** → [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) → LangGraph Lambda → Rust Lambda (retrieve) → Bedrock AgentCore/llama.cpp (generate)

---

## Repository layout

Here’s how the code and docs are organized:

```text
ultron-embeddings/
├─ crates/
│  ├─ marvel_client/       # API auth, retries, gzip, ETag cache
│  ├─ derive/              # normalize → notes with links
│  ├─ chunker/             # sentence-aware chunking
│  ├─ embedder/            # Candle embeddings (Lambda/CLI)
│  ├─ s3v_store/           # vector upsert/query
│  └─ lambda_retrieve/     # Rust Lambda handler (data plane)
├─ orchestrator/           # LangGraph app (Python Lambda, control plane)
├─ infra/
│  └─ cdk/                 # [AWS Cloud Development Kit (CDK)](../infra/README.md) stacks (S3, API Gateway, Lambdas, Layers, DynamoDB)
└─ docs/                   # deeper guides (see index)
```

---

## Orchestrator (LangGraph/LangChain)

LangGraph runs as the **control plane Lambda**:

- **Query analysis** — rewrites user text, attaches filters (e.g. series, era)  
- **Retriever tool** — calls Rust `/retrieve` Lambda for top-k chunks  
- **Prompt assembly** — wraps snippets in Ultron-style system prompt with Marvel links  
- **Generation** — routes to Bedrock AgentCore or `llama.cpp` Lambda depending on mode  
- **Streaming** — tokens stream back to the client via API Gateway  
- **Evaluation** — every request traced in LangSmith for debugging and quality  

This keeps the data plane (Rust) fast, while orchestration (Python) remains flexible.

---

## S3 and metadata

- **System metadata**: size, last-modified, storage class, ETag (S3-controlled)  
- **User metadata**: `x-amz-meta-*` pairs set at upload (immutable)  
- **Rich fields**: stored in JSONL records alongside objects  
- **Delta syncs**: rely on `ETag` and `Last-Modified`; retry with `If-Modified-Since`  
- **Discovery**: AWS S3 Metadata tables (Iceberg) can accelerate estate-wide queries, but this repo treats S3 layout as the source of truth  

Details: [`docs/state-and-storage.md`](./docs/state-and-storage.md)

---

## Docs Index

Find deep dives in `/docs`.

- **CONTRIBUTORS.md** — How we work: [Pull Request (PR) checklist](https://docs.github.com/en/pull-requests), style, tests  
- **mcp.md** — VS Code + Copilot MCP setup  
- **ingest.md** — Marvel ingestion, retries, ETag cache  
- **derive.md** — Data normalization and transformation  
- **chunk.md** — Chunking strategy  
- **embed.md** — Candle embeddings, quantization, eval  
- **retrieve.md** — Rust retriever contract  
- **state-and-storage.md** — Vector index format, DynamoDB sessions  
- **orchestrator.md** — LangGraph nodes, tools, streaming, LangSmith  
- **marvel-api.md** — Marvel API integration details  
- **settings.md** — Environment variables, overrides  

See also `infra/README.md` for CDK insights.

---

## Project guidelines

- Heavy binaries live in **Lambda Layers** (Candle, `llama.cpp`)  
- Models stored in **S3**, copied to `/tmp` on cold start  

---

## License

- **Code & derived text:** MIT Open  
- **Marvel data/images:** follow Marvel’s Terms of Use. Always attribute  

> “Change is the essential process of all existence.” — Ultron

<!-- 54654524F4E -->
