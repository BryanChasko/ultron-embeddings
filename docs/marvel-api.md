# Marvel API — Ultron Playbook (api.md)

> This file introduces core **API** concepts using the Marvel API and our Ultron use‑case.  
> See **[Concepts Covered](#concepts-covered)** if you’re new to terms like HTTP, headers, hashes, or rate limits.

---

## Friendly Neighborhood Intro to APIs

Let's refresh on the basics.  

### What’s an API?
An **API** (Application Programming Interface), in this case the one run by Marvel’s computer systems, has all the information their team has made available — data about heroes, villains, comics, stories, series, and events.  
- The API defines what you can ask for and how to ask.  
- When you send a request, Marvel’s systems prepare the data and send back a structured response.  

You don’t need to know the inner workings of Marvel’s infrastructure. What matters is that when you follow the instructions, you’ll consistently receive the data you need.

---

### Who Are You Talking To?
When you “ask” Marvel for data, you’re contacting a **server**.  
- A server is a computer designed to always listen for requests and return responses.  
- In this playbook, Marvel’s servers are the canonical source for information about **Ultron-6** — his adamantium body, his appearances across comics, and his role in major events.

---

### What Do You Get Back?

Responses arrive in **JSON** (JavaScript Object Notation).  

- JSON is a structured text format made of simple key–value pairs.  
- It’s the most widely adopted data format on the web because it’s:  
  - **Human-readable** — you can open it in any text editor and understand it.  
  - **Machine-friendly** — every programming language has JSON parsing built in.  
  - **Lightweight** — compact enough to send quickly.  
  - **Flexible** — handles nested objects, arrays, numbers, and text without rigid schemas.  

Example response about **Ultron-6 (adamantium body)**:

```json
{
  "id": 1009685,
  "name": "Ultron-6",
  "chassis": "Adamantium Alloy",
  "firstAppearance": "Avengers #66 (1969)",
  "notableStory": "Attempted nuclear strike on New York City",
  "powers": ["Superhuman strength", "Durability", "Technopathy"]
}
```

Because the structure is predictable, you always know where to look for fields like `chassis`, `powers`, or `notableStory` without guesswork.

---

### What About JSONL?

You may also see **JSONL** (JSON Lines). This format stores **one JSON object per line** in a text file:  

```json
{"id": 1, "name": "Ultron-6", "chassis": "Adamantium Alloy", "type": "character"}
{"id": 2, "title": "Avengers #66", "type": "comic"}
{"id": 3, "event": "Ultron vs. Vision", "type": "story arc"}
```

Why it matters:  
- **Streaming-friendly** — process records line by line without loading everything at once.  
- **Vector databases & LLMs** — JSONL is the standard input for embeddings. Each record (a character, a comic, or an event) is independent and can be embedded, indexed, and retrieved on its own.  
- **Consistency** — every line has the same schema, making large-scale processing stable and efficient.  

JSON/JSONL has become the **lingua franca of APIs and AI pipelines** — simple and adaptable enough to last generations.

---

## Marvel Authentication (What the Server Expects)

Marvel requires three pieces on every request:

- **ts** — a timestamp you generate for this request.  
- **apikey** — your **public** key (safe to send).  
- **hash** — an **MD5** of `ts + privateKey + publicKey` proving you know the private key without sending it.

Example shell helper to create the hash:

```sh
ts=$(date +%s)
hash=$(echo -n "${ts}${MARVEL_PRIVATE_KEY}${MARVEL_PUBLIC_KEY}" | md5sum | cut -d' ' -f1)
```

Example environment variables file (where those keys come from):

```dotenv
MARVEL_PUBLIC_KEY=pk_...
MARVEL_PRIVATE_KEY=sk_...
```

We keep secrets out of the repo. Store them securely and export them when needed. see settings.md for guidance.

---

## Endpoints We Actually Use

- `GET /v1/public/characters?name=Ultron` — **find Ultron’s character record** (ID, urls, thumbnail, modified).  
- `GET /v1/public/characters/{characterId}/comics` — **Ultron appearances** in comics.  
- `GET /v1/public/characters/{characterId}/stories` — **story arcs** related to Ultron.  
- `GET /v1/public/characters/{characterId}/events` — **crossovers** where Ultron shows up.  
- `GET /v1/public/comics?characters={id}` — **comics featuring Ultron**.  
- `GET /v1/public/series?characters={id}` — **series** associated with Ultron.

Useful query params:  
- `limit` (default 20, max 100) + `offset` — page through all results without blowing the daily cap.  
- `orderBy=-modified` — newest records first.  
- `modifiedSince=YYYY-MM-DD` — only what changed since a given date.

---

## Rate Limits and Reliability

- **Daily cap** — 3,000 requests per key; start with `limit=1` while testing.  
- **429 Too Many Requests** — back off and retry.  
- **5xx Server Errors** — retry with jitter, then skip if persistent.  
- **Compression** — use `Accept-Encoding: gzip`.  
- **Connection reuse** — enable keep-alive for efficiency.

---

## Delta Syncs Without Wasting Calls

Only pull what changed:  
- **If-Modified-Since / Last-Modified** — request only newer data.  
- **ETag / If-None-Match** — let the server tell you if content has changed.

---

## Shaping Responses for the Pipeline

Normalize Marvel’s JSON into predictable schemas. Keep raw pages for provenance; write trimmed JSONL for downstream use.

Ultron Character Record (trimmed):  
```json
{
  "type": "character",
  "id": 1009685,
  "name": "Ultron-6",
  "chassis": "Adamantium Alloy",
  "modified": "2024-07-19T18:30:34-0400",
  "urls": [{"type":"detail","url":"https://marvel.com/characters/1009685/ultron"}],
  "thumbnail": {"path": "...", "extension": "jpg"}
}
```

Comic (trimmed):  
```json
{
  "type": "comic",
  "id": 123456,
  "title": "Avengers #66",
  "pageCount": 32,
  "modified": "2024-06-01T12:00:00-0400",
  "urls": [{"type":"detail","url":"https://marvel.com/..."}],
  "characters": [1009685]
}
```

---

## Data Flow at a Glance

```mermaid
flowchart LR
    A[HTTP Request<br/>Marvel API] --> B[JSON Response]
    B --> C[Normalize to JSONL]
    C --> D[Vector DB<br/>(Embeddings, LLMs)]
```

> _Ultron:_ “Requests become responses. Responses become lines. Lines become vectors. And vectors… become power.”

---

## Concepts Covered

- **API** — structured way to send requests and get responses.  
- **Server** — computer that receives and returns responses.  
- **HTTP methods** — GET (read), POST (create), PUT/PATCH (update), DELETE (remove).  
- **Query params** — refine requests with `?name=value`.  
- **Headers** — metadata for requests/responses.  
- **Status codes** — `2xx` success, `4xx` client error, `5xx` server error.  
- **Rate limiting** — request budget per time window.  
- **ETag / Last-Modified** — detect if data changed.  
- **MD5 hash** — prove you hold the private key without sending it.  
- **JSON / JSONL** — structured data formats for APIs and pipelines.

---

## References

- Marvel developer portal — https://developer.marvel.com/  
- Internal: Rust client implementation in `crates/marvel_client/`  
- Internal: `docs/ingest.md` for ingest pipeline integration  

`54654524F4E` → **ULTRON**
