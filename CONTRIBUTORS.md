# CONTRIBUTORS.md

## Project philosophy

* **Empathy & clarity:** Communicate like a human. Brief, specific, action‑oriented.
* **Bias for action:** Prefer proposing and applying patches. If you must pause, state the exact blocker and the next command to run.
* **Rust‑first:** Application logic lives in Rust CLIs; shell is for orchestration only.
* **Docs > code:** Update docs alongside code in the same PR.

## Branching & commits

* Default branch: **`main`**.
* Use **Conventional Commits** (e.g., `feat:`, `fix:`, `docs:`, `refactor:`).
* Keep PRs small and end‑to‑end (ingest → derive → chunk → embed → index → demo where feasible).

## Code standards

* `rustfmt`/`clippy` clean.
* Every crate exposes `--help` with examples.
* Error messages must include **actionable remediation**.
* Any new data written to disk must include a **short README** in its folder explaining format and retention policy.

## Marvel API compliance checklist (for any ingest changes)

* [ ] Auth includes `ts`, `apikey`, `hash=md5(ts+private+public)`
* [ ] Use `Accept-Encoding: gzip`
* [ ] Use **ETag** caching (`If-None-Match`) and support 304
* [ ] Observe `modifiedSince` where available
* [ ] Page with `limit=100` + `offset`
* [ ] Back off on `429`; rejected calls still count against quota
* [ ] Persist `attributionText`/`attributionHTML` with results
* [ ] When displaying more than a title + small thumb, **link each item** to its `urls[]`
* [ ] Use documented **image variants**; do not rehost full‑res art

## Secrets & data handling

* Never commit API keys or raw dumps with PII.
* `.env` is required locally; CI uses GitHub Secrets.
* `data/raw/` is cached JSON (subject to Marvel terms); prune with care and keep within repo size limits (prefer local storage or an object store for large crawls).

## Testing & evals

* Unit tests for client (auth, paging, ETag) and schema validation.
* Snapshot tests for derivation (stable, short summaries).
* Eval harness for retrieval: link‑precision\@k, disambiguation cases.

## GitHub Copilot pairing preferences

Paste the following into `.github/copilot-instructions.md` (or an issue) so agents behave consistently:

> **Preference:** Assume a bias for action. Propose a concrete patch and proceed. If a human confirmation is absolutely required, end with: *“I will apply this patch. Say ‘yes’ to continue.”* Avoid generic options; provide the exact commands you will run.

## PR checklist

* [ ] Lints pass (`cargo fmt`, `cargo clippy`)
* [ ] Adds/updates docs
* [ ] Marvel compliance checklist reviewed
* [ ] Adds or updates tests/evals when applicable
* [ ] Demo still serves attribution and valid links

## Issue labels (suggested)

* `ingest` · `schema` · `derivation` · `chunking` · `embeddings` · `index` · `retriever` · `eval` · `docs` · `compliance` · `good first issue`

## Contact & attribution

* Attribution line required in any UI or API response containing Marvel data: `Data provided by Marvel. © MARVEL`
* External links must use one of the `urls[]` from the API.

Thanks for contributing—now ship something Ultron‑worthy. 🦾
