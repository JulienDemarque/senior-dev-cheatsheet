# ML, LLMs & agents

**RAG**, embeddings, tool-using agents, and provider APIs—patterns and minimal snippets (always validate production code with current vendor docs).

## Vector database

Stores embedding vectors + metadata; queries return nearest neighbors (cosine, dot product, L2). ANN indexes (HNSW, IVF) trade recall vs speed.

### Example (query shape — conceptual)

```json
POST /index/query
{ "vector": [0.01, -0.2, ...], "top_k": 10, "filter": { "org_id": "acme" } }
```

## Pinecone

Hosted **vector database**: no index tuning cluster ops for many teams; pay for storage + QPS tiers.

## RAG (retrieval-augmented generation)

Retrieve chunks → build prompt with citations → LLM answers. Quality levers: chunking strategy, hybrid retrieval, **reranking**, and eval sets.

### Example (prompt skeleton)

```text
You are a support bot. Use ONLY the following context.

Context:
---
{retrieved_chunks}
---

Question: {user_query}
If the answer is not in the context, say you don't know.
```

## Embeddings

Fixed-size vectors from an encoder model; “similar meaning → nearby vectors” approximately. Normalize vectors if using cosine dot on unit spheres.

### Example (OpenAI embeddings — illustrative)

```python
from openai import OpenAI
client = OpenAI()
resp = client.embeddings.create(model="text-embedding-3-small", input="hello world")
vec = resp.data[0].embedding
```

## Hybrid retrieval

Combine dense ANN with sparse lexical (BM25) or metadata filters—improves recall on rare tokens and proper nouns.

## Reranking

Cross-encoder or small LLM scores `(query, candidate)` pairs; run on top-50 from ANN, feed top-5 to the big model.

## DSL

Constrained language for queries or tools (SQL subset, Lucene query string, JSON filter grammar)—limits prompt injection blast radius vs raw string pass-through.

## Structured outputs

Force JSON shape, enums, or tool calls so downstream code can parse without regex. Combine **JSON schema** with constrained decoding or repair loop.

### Example (JSON schema fragment)

```json
{
  "type": "object",
  "required": ["answer", "confidence"],
  "properties": {
    "answer": { "type": "string" },
    "confidence": { "type": "number", "minimum": 0, "maximum": 1 }
  }
}
```

## JSON schema

Describe JSON types for validation (AJV, pydantic `json_schema`, OpenAI `response_format`). Version schemas like APIs.

## MCP server

Model Context Protocol: stdio or HTTP server exposing tools/resources to clients (Cursor, Claude Desktop). Typed tool calls instead of ad-hoc scraping.

## Agentic workflow

Planner + tool loop + memory; risks: runaway loops, cost, data exfiltration. Add budgets, allowlists, and **human-in-the-loop** for destructive tools.

## Human-in-the-loop

Approval UI or ticketing for sensitive actions; also produces labeled data for **evaluation loop** improvements.

## Evaluation loop

Offline datasets (golden Q&A), online A/B, LLM-as-judge with rubrics—tied to CI for regressions on prompts/models.

## LiteLLM

Proxy/SDK routing to many providers with unified interface, fallbacks, budgets, and logging hooks.

### Example (drop-in style)

```python
from litellm import completion
resp = completion(model="gpt-4o-mini", messages=[{"role": "user", "content": "Hi"}])
```

## LlamaIndex

Indexing pipelines, data connectors, query engines over docs—**RAG**-centric ergonomics in Python.

## LangChain

Composable chains, tools, memory abstractions; large integration surface—pin versions; test chains in CI.

## LangGraph

State machine / graph nodes for cycles (retry branches, human approval gates) atop LangChain primitives.

## OpenAI API

Chat Completions / Responses APIs, embeddings, images, audio. Features (tools, JSON mode, caching) vary—read release notes per model.

## Structured generation

Provider-native JSON mode or tool calls; locally, constrained decoding libraries reduce invocations vs “please output JSON” prompts alone.

## Prompt caching

Reuse long static prefix KV blocks across requests for lower latency/cost when system prompt and docs are stable.

## Speculative decoding

Small draft model proposes tokens; large model verifies batches—throughput win when draft aligns with target distribution.
