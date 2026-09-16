Context: This is Cleo, a multi-agent assistant built on Microsoft Agent Framework (Python), Azure OpenAI, Azure AI Search, and Cosmos DB. An external ADO pipeline (core-ccoe-ccoebot-data-pipeline) already embeds SharePoint KB pages and writes them to an Azure AI Search index. I currently expose that index through an MCP tool (search_sharepoint) on a separate MCP server. I want to remove the MCP dependency and do retrieval in-process.

Index contract (do not change; treat as external):
- Index: sharepoint-index on https://gmf-eastus2-devtest-ccoebot-search-service.search.windows.net
- Fields: id (key), content (searchable text), embedding (Collection(Single), 3072-dim, HNSW), title, sourceurl, file_name, parent_id, chunk_index, sharepoint_id, timestamp, document_data
- Embeddings were produced with Azure OpenAI deployment text-embedding-3-large (3072-dim). Query embeddings MUST use the same deployment.
- The pipeline deletes and recreates the index on every run (~7 min window where it may be empty or missing). Treat "index not found" and zero hits as normal, non-fatal states.

Build:
1. A `SharePointRetriever` module (services/retrieval/sharepoint.py):
   - Config from env/settings: SEARCH_ENDPOINT, SHAREPOINT_INDEX_NAME, EMBEDDING_DEPLOYMENT, TOP_K (default 6). No secrets; auth via DefaultAzureCredential (managed identity in Azure, az login locally).
   - Lazily construct SearchClient and the embeddings client on first use, never at import or app startup. No network calls in health checks that only inspect constants — a health probe, if added, must actually hit the index with a cheap query.
   - `retrieve(query: str, top_k: int | None = None) -> list[Chunk]` doing hybrid search (BM25 on content + vector on embedding via VectorizedQuery), semantic ranker if the service has a semantic configuration, otherwise skip. Return Pydantic `Chunk` objects with content, title, sourceurl, chunk_index, parent_id, score.
   - Dedupe by parent_id (keep highest-scoring chunk per page, then fill remaining slots), and truncate total content to a configurable token budget.
   - On index-not-found, throttling, or auth errors: log at WARNING with the error class, return an empty list, and set a `RetrievalStatus` (ok / empty / unavailable) the caller can read. Do not raise into the agent loop.

2. Orchestration: on the knowledge-base / fulfillment path (find where the router dispatches KB-style questions), call the retriever before the agent runs and inject chunks into the agent's context as a clearly delimited "Reference documents" block with [n] markers and a source list (title + sourceurl). Instruct the agent to cite [n] and to say when the references don't cover the question. If status is `unavailable`, tell the agent the knowledge base is temporarily unavailable so it doesn't guess.

3. Fallback tool: register a native Agent Framework function tool `search_sharepoint(query: str, top_k: int = 5)` on the same agent, wrapping the same retriever, so the model can re-query when the pre-retrieval pass is insufficient. Keep the tool signature Pydantic-validated. Remove the MCP-based search_sharepoint tool and its registry/allowlist entry, and note the removal in a short ADR (docs/adr/) explaining that the MCP policy boundary no longer applies to this data source and why that's acceptable (read-only index, first-party data, managed identity RBAC).

4. Tests (pytest): unit tests with a mocked SearchClient covering: hybrid query construction uses the configured embedding deployment; dedupe by parent_id; empty-result and index-not-found paths return [] with the right status; context injection formats citations correctly; the retriever is not constructed at import time. Add a contract test that asserts the expected index field names against a checked-in index_schema.json so a pipeline-side rename fails CI.

Constraints: keep changes minimal and localized; follow the repo's existing settings/config pattern and logging; don't touch the guardrail gate; don't add new secrets or connection strings. Before writing code, list the files you'll touch and any assumptions about where the KB path lives, then proceed.

<div align="center">

```
   ┌──────────────────────────────────────────┐
   │  $ whoami                                  │
   │  > chirag dewan                            │
   │  $ ./objective.sh                          │
   │  > breaking AI. responsibly.               │
   └──────────────────────────────────────────┘
```

</div>

```python
class Chirag:
    role   = "AI security researcher"
    focus  = "red teaming LLMs, agents, and the systems on top"
    method = "attacker mindset + reproducible evals"
```

### currently hunting

```diff
+ indirect prompt injection in agentic systems
+ tool-use exploitation & privilege escalation via AI
+ jailbreaks that survive RLHF
- benchmarks that pretend models are safe
```

```
prev  ▸ offensive security @ raytheon BBN — zero-days, ICS, PoCs
now   ▸ AI security & red teaming · dallas, tx
```

<!-- selected work — uncomment each line once the artifact is shipped + validated
### selected work

- **MCP-Poison-Bench** — agentic tool-poisoning benchmark + client-side defense · [repo](#) · [writeup](#)
-->

<div align="center">

reach ▸ [chirag0728@gmail.com](mailto:chirag0728@gmail.com) · [cdewan.me](https://cdewan.me)

*"the model is not the system. the system is the attack surface."*

</div>
