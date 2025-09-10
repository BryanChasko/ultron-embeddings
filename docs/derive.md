# Derive — Canonical Entities + Notes

--

## Purpose

Take **raw Marvel API JSON Lines (JSONL)** from the ingest step and turn it into:
- **Canonical entities** (character, comic, series, event, story) with a consistent shape.
- **Relation edges** (e.g., `character → comic`, `comic → series`).
- **Derived notes**: short summaries and relationship hints that link back to Marvel URLs.
- **Ready‑for‑chunking JSONL**, with clean text fields and stable metadata.

Why this exists: once the raw is frozen in S3, we can re‑run derive any time without touching Marvel again. It keeps iteration cheap and safe on the request cap.

See also:
- [`ingest.md`](./ingest.md) — how the raw JSONL lands in S3
- [`marvel-api.md`](./marvel-api.md) — auth, endpoints, etiquette
- [`chunk.md`](./chunk.md) — how text is split for embeddings
- [`state-and-storage.md`](./state-and-storage.md) — S3 layout, checksums, and indexing

---

## Inputs → Outputs

**Input** (from ingest):
- JSONL files under `s3://<bucket>/raw/YYYY/MM/DD/*.jsonl`
- Each line is a single Marvel API object (as returned).

**Output** (from derive):
- JSONL under `s3://<bucket>/derived/YYYY/MM/DD/*.jsonl`
- One line per canonical record (see schema below).
- A **deterministic** record key for each entity (`source_id`, `entity_type`).
- Optional `edges/*.jsonl` with relation rows (small, flat records).

A single pass can emit two streams:
- `derived/entities-*.jsonl`
- `derived/edges-*.jsonl`

---

## Canonical schema (entities)

All entities share a base shape and then add type‑specific fields.

**Base fields (shared):**
- `entity_type`: `"character" | "comic" | "series" | "event" | "story"`
- `source`: `"marvel"`
- `source_id`: Marvel numeric or composite ID (stringified)
- `title`: compact human title (for character, use `name`)
- `description`: short text (may be empty; never invent copy)
- `modified`: ISO‑8601 string from Marvel
- `urls`: array of `{ type, url }` (keep all types; prefer `detail` when linking out)
- `thumbnail`: `{ path, extension }` when present
- `attribution`: `"Data provided by Marvel. © 2025 MARVEL"`
- `schema_version`: integer, starts at `1`
- `ingest_run_id`: identifier of the raw batch this row came from
- `record_checksum`: content hash of the canonical record body (for idempotence)

**Characters add:**
- `aliases`: string[] (if available; else empty)
- `counts`: `{ comics, series, stories, events }`

**Comics add:**
- `issue_number`: number
- `page_count`: number | null
- `prices`: `{ type, price }[]`
- `series_id`: string | null
- `onsale_date`: ISO‑8601 | null

**Series add:**
- `start_year`: number | null
- `end_year`: number | null

**Events add:**
- `start`: ISO‑8601 | null
- `end`: ISO‑8601 | null

**Stories add:**
- `type_hint`: `"cover"` | `"interiorStory"` | string | null

> JSON Schema files live under `docs/schemas/*.schema.json`. The derive binary validates each record against its schema before writing. See examples below.

---

## Derived notes (short, linked, factual)

**Goal:** add compact, factual nuggets that make retrieval crisp without quoting large blocks.

- Field: `notes`: string[] (each note ≤ 240 chars)
- Each note **must** reference at least one `urls[].url` from the entity (inline or as a sibling `note_links: string[]`).
- No long quotes. Summarize. Keep it neutral and specific.
- Examples:
  - `"Ultron first appeared opposing the Avengers; creator: Hank Pym (see detail)."`
  - `"Omega Ultron arc intersects West Coast Avengers (see series detail)."`

If a note is speculative or spans entities, prefer placing it on the **edge** (see below).

---

## Relations (edges)

Edges make cross‑entity queries cheap and explicit.

**Edge record shape:**
- `edge_type`: `"character_appears_in_comic" | "comic_part_of_series" | "character_in_event" | "story_in_comic" | ...`
- `from`: `{ entity_type, source, source_id }`
- `to`: `{ entity_type, source, source_id }`
- `links`: string[] (Marvel URLs supporting the relation)
- `ingest_run_id`, `schema_version`, `record_checksum`

Recommended edges we emit:
- `character_appears_in_comic` (Ultron → comic)
- `comic_part_of_series` (comic → series)
- `character_in_event` (Ultron → event)
- `story_in_comic` (story → comic)

---

## Determinism, replays, and versions

- **Pure function:** canonical output is a pure function of raw input (plus the code version). No network calls here.
- **Stable IDs:** `source_id` is always Marvel’s ID; never re‑key.
- **`schema_version` bump:** only when the canonical fields change meaning or names.
- **Idempotence:** we compute `record_checksum` (e.g., SHA‑256 over a stable JSON representation). Writers upsert by `(entity_type, source_id)` and skip if checksum unchanged.
- **Replay friendly:** you can delete `derived/*` for a date and re‑run derive; results will match bit‑for‑bit given the same code version.

---

## Where to run

- **Local development:** best for evolving the shape, writing tests, and inspecting examples quickly.
  - You can point the derive binary at `./target/data/raw/` and write to `./target/data/derived/` while iterating.
- **Lambda (batch job):** good when you want scheduled, reliable production runs right after ingest.
  - The function reads `s3://.../raw/...`, writes `s3://.../derived/...`, and logs basic counts.

Both paths use the exact same Rust binary. The Lambda is just a wrapper/driver.

---

## CLI usage

Local example:
```bash
cargo run -p derive -- \
  --in ./target/data/raw/2025/09/09 \
  --out ./target/data/derived/2025/09/09 \
  --schema-version 1 \
  --emit-edges
```

S3 example (uses `AWS_REGION` and instance role/credentials):
```bash
cargo run -p derive -- \
  --in s3://ultron-embeddings-123456789012/raw/2025/09/09 \
  --out s3://ultron-embeddings-123456789012/derived/2025/09/09 \
  --emit-edges
```

Flags:
- `--in`, `--out`: input and output prefixes (local path or `s3://`).
- `--emit-edges`: also write `edges-*.jsonl`.
- `--schema-version`: override default if you’re testing a new schema.
- `--fail-fast`: stop on first schema error (default: continue and report).

---

## Examples

**Character (Ultron):**
```json
{
  "entity_type": "character",
  "source": "marvel",
  "source_id": "1009685",
  "title": "Ultron",
  "description": "A highly advanced AI with a fixation on human extinction.",
  "modified": "2024-07-19T18:30:34-0400",
  "urls": [{"type":"detail","url":"https://marvel.com/characters/1009685/ultron"}],
  "thumbnail": {"path": "https://i.annihilus/ultron", "extension": "jpg"},
  "aliases": [],
  "counts": {"comics": 120, "series": 25, "stories": 180, "events": 8},
  "notes": [
    "Created by Hank Pym; core Avengers adversary (see detail)."
  ],
  "schema_version": 1,
  "ingest_run_id": "2025-09-09T00:00:00Z-ultron",
  "record_checksum": "sha256:..."
}
```

**Edge (Ultron appears in a comic):**
```json
{
  "edge_type": "character_appears_in_comic",
  "from": {"entity_type":"character","source":"marvel","source_id":"1009685"},
  "to": {"entity_type":"comic","source":"marvel","source_id":"123456"},
  "links": ["https://marvel.com/characters/1009685/ultron","https://marvel.com/comics/issue/123456"],
  "schema_version": 1,
  "ingest_run_id": "2025-09-09T00:00:00Z-ultron",
  "record_checksum": "sha256:..."
}
```

---

## Validation

- Each canonical record is validated against its JSON Schema (`docs/schemas/*.schema.json`).
- Failures are logged with the offending line and field path.
- When `--fail-fast` is off (default), invalid rows are skipped but written to `derived/_rejects/` for inspection.

---

## Error handling

- **Empty descriptions**: keep them empty; do not synthesize text.
- **URL sets**: pass through all URL types from Marvel; pick `detail` as default when linking out.
- **Date parsing**: invalid dates are kept as original strings and flagged in a `warnings[]` field.
- **Unexpected shapes**: stash original raw object under `raw_debug` only when `--debug` is set (to avoid bloat).

---

## Attribution

Top‑level attribution is enough for outputs and docs surfacing Marvel fields:

```
Data provided by Marvel. © 2025 MARVEL
```

---

## Next

Move to [`chunk.md`](./chunk.md) to split canonical text into sentence‑aware chunks ready for embeddings.
