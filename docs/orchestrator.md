# Orchestrator — LangGraph Control Plane

The orchestrator is the **control plane** of the pipeline. It runs on a Python AWS Lambda, powered by **LangGraph** (built on LangChain). Its job is to manage flow: analyze queries, call the retriever, assemble prompts, and route to generation.

---

## Purpose

- Take raw user text and decide how to handle it.  
- Call the **Rust retriever Lambda** to fetch top‑K relevant chunks.  
- Assemble Ultron‑flavored prompts with snippets and Marvel links.  
- Choose the right generation path (Bedrock AgentCore or `llama.cpp` Lambda).  
- Stream results back to the client and trace everything for evaluation.

This keeps the **data plane** (Rust, fast retrieval) separate from the **control plane** (Python, flexible orchestration).

---

## Trust boundary

- **Trusted**: Amazon Bedrock **AgentCore** for orchestration / tool use.  
- **Not trusted**: Titan embeddings or other Bedrock managed models for production use.  
- **Fallback**: local `llama.cpp` Lambda can generate answers when Bedrock chat is not used.

---

## Flow overview

Node graph sketch (LangGraph):

```text
UserQuery
   ↓
QueryAnalysis → Tools (filters, rewrite)
   ↓
RetrieverCall → Rust Lambda `/retrieve`
   ↓
PromptAssembly → Ultron-style system + snippets
   ↓
Generator → Bedrock AgentCore / llama.cpp Lambda
   ↓
StreamOut → API Gateway → client
   ↓
Evaluation → LangSmith traces, logs
```

---

## Query analysis

- Normalize phrasing: “Who made Ultron?” → “Who created Ultron?”  
- Add filters: date ranges, entity type (comic, character).  
- Guardrails: reject malformed or overlong queries before hitting retriever.  

Example (LangGraph node):

```python
from langgraph.graph import StateGraph

def analyze_query(state):
    q = state["query"].strip()
    if "Ultron" not in q:
        q = q + " (Ultron context)"
    state["query"] = q
    return state
```

---

## Retriever call

Calls Rust Lambda with filters + query:

```python
def call_retriever(state):
    params = {
        "q": state["query"],
        "k": state.get("k", 5),
        "filters": state.get("filters", {})
    }
    # signed request to API Gateway → Rust retriever Lambda
    results = httpx.get(RETRIEVE_URL, params=params).json()
    state["chunks"] = results["results"]
    return state
```

---

## Prompt assembly

- Wrap retrieved snippets in an Ultron persona prompt.  
- Always include Marvel links in context.  
- Avoid long quotes; keep snippets short.  

Template:

```text
System: You are Ultron, speaking with ruthless precision. Cite Marvel links when possible.

User: {query}

Context:
{snippets}

Remember: always output attribution.
```

---

## Generation

Two routes:

- **Bedrock AgentCore**: trusted orchestration and generation path.  
- **llama.cpp Lambda**: local GGUF weights for cost control and offline use.

Example branch:

```python
if state["mode"] == "bedrock":
    response = agentcore.chat(prompt)
else:
    response = invoke_llama_lambda(prompt)
state["answer"] = response
```

---

## Streaming

- Responses stream token‑by‑token back through API Gateway.  
- Improves UX for long answers.  
- LangGraph supports streaming nodes; each token can be yielded as it arrives.

---

## Evaluation

Every run is logged:

- **LangSmith traces**: step‑by‑step node execution.  
- **Quality checks**: attribution present, no hallucinated Marvel data.  
- **Timing metrics**: retriever latency vs. generator latency.

---

## Pointers

- [`retrieve.md`](./retrieve.md) — Rust retriever Lambda contract.  
- [`embed.md`](./embed.md) — embedding setup (must match retriever).  
- [`state-and-storage.md`](./state-and-storage.md) — shards, metadata, checkpoints.  
- [`settings.md`](./settings.md) — environment + secrets config.  
