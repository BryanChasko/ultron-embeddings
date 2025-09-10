# Chunk — Sentence-Aware Splitting

---

## Purpose

The **chunk** step breaks normalized + derived Marvel records into smaller, 
search-friendly text spans. Chunking is essential because vector search and 
LLM context windows perform best on compact, semantically complete passages.

Outputs land in `s3://<bucket>/chunks/YYYY/MM/DD/*.jsonl`.

---

## Why chunk?

- **Vector search accuracy**: Smaller spans reduce irrelevant matches.  
- **LLM context limits**: Foundation models can only handle a finite number 
  of tokens. Chunking ensures we pass in manageable, high-signal text.  
- **Efficient updates**: When only part of a comic description changes, 
  re-chunking is cheaper than reprocessing the whole dataset.

---

## Strategy

- **Target size**: ~280 characters per chunk.  
- **Overlap**: 40 characters (ensures continuity across split boundaries).  
- **Sentence-aware**: Split on sentence boundaries first; fall back to 
  hard cuts only if needed.  
- **Metadata carry-over**: Each chunk preserves `id`, `type`, `source_url`, 
  and `modified` fields from the parent record.  

Example derived record:

```json
{
  "type": "comic",
  "id": 123456,
  "title": "West Coast Avengers (2024) #10",
  "description": "Ultron returns. Heroes must unite. Battle spans the multiverse.",
  "urls": ["https://marvel.com/..."],
  "modified": "2024-06-01T12:00:00-0400"
}
```

Becomes chunked:

```json
{
  "parent_id": 123456,
  "chunk_id": "123456-001",
  "text": "Ultron returns. Heroes must unite.",
  "overlap_with": null
}
{
  "parent_id": 123456,
  "chunk_id": "123456-002",
  "text": "Battle spans the multiverse.",
  "overlap_with": "123456-001"
}
```

---

## Where to run

- **Local development**: Run the `chunker` crate for fast iteration. Useful for 
  tuning chunk size, overlap, and sentence rules.  

- **Lambda (production)**: Event-driven Lambda transforms newly derived objects 
  from S3 into chunked JSONL. Light compute, no external calls.

---

## S3 layout

```
s3://ultron-embeddings-<account-id>/
└── chunks/
    └── 2025/
        └── 09/
            └── 09/
                comics-00001.jsonl
                stories-00001.jsonl
```

---

## Error handling

- **Empty text**: Skip record, log warning.  
- **Oversized fields**: Force-split into smaller spans.  
- **Encoding issues**: Normalize to UTF-8.  
- **Downstream contract**: Always produce valid JSONL lines, even if a chunk is empty.

---

## Local run example

```sh
cargo run -p chunker --   --in ./target/data/derived/   --out ./target/data/chunks/
```

Output: `./target/data/chunks/*.jsonl`

---

## Lambda run

- **Trigger**: EventBridge rule (e.g., after derive step).  
- **Handler**: Rust Lambda that reads derived objects from S3 and writes chunked JSONL back to S3.  
- **State**: DynamoDB checkpoint optional (not required if upstream derive is stable).  

---

## Best practices

- Keep chunks semantically meaningful (avoid cutting in the middle of names).  
- Tune chunk size for balance: smaller = higher recall, larger = higher context.  
- Ensure overlap to avoid missing cross-boundary references.  

---

Next step: see [`docs/embed.md`](./embed.md) for generating embeddings from chunked data.
