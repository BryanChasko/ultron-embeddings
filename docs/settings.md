# Settings

Central place for environment and runtime knobs. This assumes your `.env` **lives outside this repo** with tighter permissions (your setup under `~/Code/Projects/Marvel-API-Private/secrets/marvel-api.env`).

---

## What lives where

- **Local secrets (preferred for dev):** your isolated `.env` file, e.g.  
  `/Users/<you>/Code/Projects/Marvel-API-Private/secrets/marvel-api.env`
- **AWS Secrets Manager (optional/prod):** store the same keys as a JSON secret.
- **VS Code tooling (optional):** reference the external `.env` in `launch.json` / `tasks.json` so terminals and run/debug sessions inherit the vars.

The app reads its config from **process environment variables**. Nothing under `docs/` or source control should contain real secrets.

---

## Minimum variables

Marvel API (see `docs/marvel-api.md` for how these are used):

```dotenv
MARVEL_PUBLIC_KEY=pk_...
MARVEL_PRIVATE_KEY=sk_...
MARVEL_RATE_BUDGET=1000
MARVEL_TIMEOUT_MS=20000
MARVEL_USER_AGENT=ultron-embeddings/0.1.0
```

AWS + storage:

```dotenv
AWS_REGION=us-east-1
S3_BUCKET=ultron-embeddings-<account-id>
CHECKPOINT_TABLE=ultron-ingest-checkpoints
```

Embedding/retrieval defaults (see `docs/embed.md` and `docs/retrieve.md`):

```dotenv
EMBED_MODEL_ID=bge-small-en-v1.5
EMBED_DIMS=384
EMBED_BATCH_SIZE=32
```

Orchestrator knobs (see `docs/orchestrator.md`):

```dotenv
ORCH_MAX_TOKENS=512
ORCH_MODEL_ROUTE=llamacpp   # one of: agentcore|llamacpp
```

---

## Using your external .env (zsh)

Your file is outside the repo (good). Source it before you run anything that needs the vars.

### One‑off in a terminal tab

```zsh
source ~/Code/Projects/Marvel-API-Private/secrets/marvel-api.env
```

### Persistent helper in `~/.zshrc`

Add a tiny function so you can type `use_marvel_env` in any new shell:

```zsh
use_marvel_env() {
  local f="$HOME/Code/Projects/Marvel-API-Private/secrets/marvel-api.env"
  if [ -f "$f" ]; then
    set -a             # export all sourced vars
    source "$f"
    set +a
    echo "env loaded ✓"
  else
    echo "not found: $f" >&2
    return 1
  fi
}
```

Then run:

```zsh
use_marvel_env
```

### Optional: `direnv` without committing secrets

If you use [`direnv`](https://direnv.net/), create `.envrc` **in your repo** that just **references** the external file (no secrets copied):

```bash
# .envrc (do NOT commit an inline secret)
dotenv "$HOME/Code/Projects/Marvel-API-Private/secrets/marvel-api.env"
```

Run `direnv allow` once. Now any shell in the project directory inherits the env.

---

## VS Code integration (no secrets in repo)

### Debug / Run (launch.json)

Point `envFile` at the external path so debug sessions pick up your vars:

```jsonc
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Cargo (marvel_client)",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/target/debug/marvel_client",
      "args": ["--character", "Ultron", "--modified-since", "2020-01-01"],
      "envFile": "/Users/<you>/Code/Projects/Marvel-API-Private/secrets/marvel-api.env"
    }
  ]
}
```

For tasks:

```jsonc
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Ingest (local)",
      "type": "shell",
      "command": "cargo run -p marvel_client -- --character Ultron --modified-since 2020-01-01",
      "options": {
        "envFile": "/Users/<you>/Code/Projects/Marvel-API-Private/secrets/marvel-api.env"
      }
    }
  ]
}
```

> If an extension doesn't support `envFile`, use the zsh `use_marvel_env` function before launching VS Code (`code .`) so the window inherits the env from your shell.

---

## AWS Secrets Manager (prod‑ish option)

Keep secrets server‑side and inject them at runtime.

### Store the secret (once)

```bash
aws secretsmanager create-secret   --name ultron/marvel-api   --secret-string '{
    "MARVEL_PUBLIC_KEY":"pk_...",
    "MARVEL_PRIVATE_KEY":"sk_...",
    "MARVEL_RATE_BUDGET":"1000",
    "MARVEL_TIMEOUT_MS":"20000",
    "MARVEL_USER_AGENT":"ultron-embeddings/0.1.0"
  }'
```

### Lambda: load on cold start (Rust sketch)

```rust
// use aws_sdk_secretsmanager::{Client as SmClient};
// let sm = SmClient::new(&aws_config::load_from_env().await);
// let resp = sm.get_secret_value().secret_id("ultron/marvel-api").send().await?;
// let blob = resp.secret_string().unwrap_or_default();
// let map: std::collections::HashMap<String, String> = serde_json::from_str(&blob)?;
// for (k, v) in map { std::env::set_var(k, v); }
```

### Local CLI helper (zsh)

```zsh
aws_sm_env() {
  local name="${1:?secret name required}"
  aws secretsmanager get-secret-value --secret-id "$name"   | jq -r '.SecretString | fromjson | to_entries[] | "export \(.key)=\(.value)"'
}
# Usage:
# eval "$(aws_sm_env ultron/marvel-api)"
```

**IAM policy** attached to the Lambda role (or your dev user) needs at least:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:<region>:<acct-id>:secret:ultron/marvel-api*"
    }
  ]
}
```

---

## CI and production notes

- Never echo secret values in logs. Prefer `***` masking if your CI supports it.
- For Lambda, pass **non‑secret** knobs via environment variables; fetch **secrets** from Secrets Manager on cold start and cache them.
- Validate mandatory keys at process start and fail fast with a clear error.

---

## Troubleshooting

- **`command not found: settings.md` in zsh** — that was the shell trying to *execute* a filename. Use `touch settings.md` to create the file, or open it in an editor: `code docs/settings.md`.
- **Vars not set inside VS Code** — ensure the `envFile` path is absolute and exists, or start VS Code from a terminal where you already ran `use_marvel_env`.
- **Lambda can’t see secrets** — check IAM (`secretsmanager:GetSecretValue`) and that the secret name/region matches.
- **Different machines** — keep the external `.env` path consistent or wrap it with the `use_marvel_env` function so only one place needs updating.
