# Cleo API — APIM backend + wiki-index retrieval (loop prompt)

You are working in the repo `core-ccoe-ccoebot-api` on a clean branch cut from commit `0b18151`. Read this whole file before doing anything. Part A is how you work. Part B is what you build. Part A overrides any instinct to be faster or more helpful.

---

## PART A — OPERATING PROCEDURE

### Autonomy
You have approval to run all three phases end to end without stopping, **except** at the two hard halts below. Do not stop to ask "shall I continue" between steps.

Hard halts (stop, report, wait):
1. **Contradiction halt** — the repo disagrees with the spec in a way that changes the design (different web framework, an existing JWT validator, no identifiable KB/fulfillment path, a settings pattern that can't hold the new keys, wiki-index schema that has no vector field). Show file paths and the contradiction, propose the smallest adaptation, and wait.
2. **Access halt** — a step needs credentials, network, or RBAC you do not have (az login, reaching the search service, fetching JWKS). Say exactly what failed and what you need. Do not stub the real call and report it as done. Continue with any later steps that do not depend on it, clearly marked as "not live-verified".

Everything else — you decide and proceed.

### The loop (run for every step in Part B)
1. **PLAN** — one sentence: what this step does, the files you'll touch, and the exact command(s) that prove it worked (a test, a grep, a git diff, a YAML parse, a live call). If you can't name a proof, split the step.
2. **ACT** — smallest change that satisfies the step. No refactors, renames, formatting sweeps, or "while I'm here" edits. Follow the repo's existing patterns for settings, logging, packaging, and tests; do not import a new pattern.
3. **VERIFY** — run the proof command(s) and the full suite (`pytest -q`). Paste real output. The number of passing tests must never go down; new tests must be seen passing.
4. **REVIEW** — read `git diff` as a hostile reviewer and answer in writing:
   - (a) Anything outside scope or against a hard constraint in Part B?
   - (b) Did I assume a field name, path, setting name, or repo pattern I didn't actually observe in a file or command output?
   - (c) Anything now executes at import or app startup that didn't before?
   - (d) Any secret, token, key, URL, or tenant id hardcoded?
   - (e) Any test weakened, skipped, or mocked so broadly it can't fail?
   If any answer is yes, fix it and re-run VERIFY before moving on.
5. **CHECKPOINT** — three lines: `done:` / `verified by:` / `next:`. Then continue.

### Failure handling
- A failing check means fix the cause, never the check.
- Same failure three times → stop, show the error verbatim, give your two best hypotheses, ask.
- Never delete, skip, or loosen a test to get green.
- If a tool or dependency is missing, add only what's needed, pinned, and say so in the phase report.

### Ground rules
- Every "verified" claim must have pasted command output in the same message.
- Do **not** edit `azure-pipelines.yml`, `Dockerfile`, `infra/`, Terraform, Bicep, Key Vault, or any Azure DevOps variable group. If you think one is needed, write it as a TODO in `docs/` and move on.
- Do **not** create `RAG_URL`, `RAG_SCOPE`, or any `CONNECTIONS__*` setting anywhere. Those belong to the web app.
- Do not touch the guardrail gate, the MCP client/gateway, the tool allowlist, or the ADR folder except where Part B says so.
- No required-variable checks, startup assertions, or validation gates. Missing config degrades safely at request time; it never crashes at import.
- Commits: exactly one per phase, titled as Part B says. Never push. Never trigger a pipeline.
- End each phase with a **Phase report**: files changed, tests added (names), commands run, what is live-verified vs mocked-only, open questions, and what a human should double-check.

Start with Phase 1, step 1.

---

## PART B — SPEC

### Context
`core-ccoe-ccoebot-api` is Cleo's API: Python, Microsoft Agent Framework, Azure OpenAI, Azure AI Search, Cosmos DB, deployed to Azure App Service with a system-assigned managed identity.

Architecture: a separate **web app** takes the user's question and POSTs it to **Azure API Management**; APIM forwards it to **this API**. This API is the backend *behind* APIM. It does not call APIM. `RAG_URL` (the APIM endpoint) and `RAG_SCOPE` (the token scope the web app requests) are web-app settings and must never appear in this repo.

History: after `0b18151`, an AI-assisted edit misread the architecture and added a pipeline gate that reads the `…infra-devtest-terraform-managed` variable group and fails unless `RAG_URL`, `RAG_SCOPE`, and `CONNECTIONS__SERVICE_CONNECTION__SETTINGS__CLIENT*` exist. That work is wrong and is removed in Phase 1.

Scope: Phase 1 cleanup, Phase 2 serve behind APIM, Phase 3 in-process retrieval from the `wiki-index` search index. **No SharePoint work anywhere.** The data-pipeline repo is not yours to touch.

---

### PHASE 1 — remove the broken APIM attempt
1. Run and paste:
   ```
   git log --oneline 0b18151..HEAD
   git status
   git diff 0b18151 --stat
   grep -rn "RAG_URL\|RAG_SCOPE\|CONNECTIONS__SERVICE_CONNECTION\|infra-devtest-terraform-managed" . --exclude-dir=venv --exclude-dir=.venv --exclude-dir=.git
   ```
2. Decide: if every commit after `0b18151` and every uncommitted change belongs to that attempt, do `git reset --hard 0b18151` (unpushed) or a single `git revert` of the range (pushed). Otherwise hand-remove only: the gate step in `azure-pipelines.yml` (the `az … --output json | python -c …` block, its `required=(…)` check, and variables/parameters/conditions/`dependsOn` that exist only for it), and every `RAG_URL` / `RAG_SCOPE` / `CONNECTIONS__*` reference introduced in code, settings modules, `.env` examples, `pyproject.toml`, `requirements*.txt`, app-settings templates. If a reference predates `0b18151`, leave it and list it in the phase report.
3. Prove: `git diff 0b18151 -- azure-pipelines.yml` is empty; `python -c "import yaml; yaml.safe_load(open('azure-pipelines.yml'))"` succeeds; the grep from step 1 returns nothing new; `pytest -q` passes at the same count as on `0b18151` (run it on `0b18151` first via `git stash`/checkout if needed to get the baseline number).
4. Commit: `Remove broken APIM validation gate`. Phase report.

---

### PHASE 2 — serve this API correctly behind APIM
**Discovery (paste findings, no code yet):** how the app is served (framework, entrypoint), existing auth/middleware, existing health endpoint, logging and correlation setup, settings module and how env vars are read, test layout and fixtures. If an existing JWT validator or correlation middleware exists, you extend it; you do not build a second one.

**Build:**
1. **Inbound token validation** on every non-health route: Entra bearer token forwarded by APIM, validated by JWKS signature (RS256), issuer (`https://login.microsoftonline.com/{TENANT_ID}/v2.0` and the v1 issuer if the repo's tokens use it), expiry with small clock skew, and audience = this API's app registration (setting `API_AUDIENCE`). Settings: `TENANT_ID`, `API_AUDIENCE`. JWKS fetched lazily on first request and cached with refresh on unknown `kid`. Missing/invalid/expired/unsigned token → 401 JSON. Missing settings → every protected request gets 401 with a logged WARNING naming the missing setting; the app still boots.
2. **Optional defense in depth**, setting `APIM_REQUIRE_HEADER` (default `false`): when true, reject requests lacking header `X-APIM-Backend-Key` matching setting `APIM_BACKEND_KEY` (value arrives via App Service Key Vault reference; the repo holds only the setting name and a placeholder in `.env.example`). Constant-time compare. Missing header → 403 JSON.
3. **Correlation**: read `traceparent` and APIM's request id header (`Ocp-Apim-Trace-Id` / `X-Request-Id`, whichever exists — check headers defensively), attach to request context, include on every log line for that request, echo `X-Request-Id` in responses.
4. **Health**: unauthenticated `GET /health` (keep the existing path if one exists) that performs one cheap real check per downstream dependency (Azure OpenAI reachability, Cosmos, Search) with a short timeout and returns `{"status": "ok"|"degraded", "checks": {...}}`. Must not just inspect constants or config.
5. **Errors**: stable JSON error shape `{"error": {"code": ..., "message": ..., "request_id": ...}}`; correct status codes (401/403/429/5xx); never include stack traces, tokens, or header values in the body.
6. **Tests** (pytest, no network; mock JWKS and dependencies): valid token accepted; expired, wrong-audience, wrong-issuer, unsigned, and missing token → 401; `APIM_REQUIRE_HEADER=true` + missing/wrong header → 403; request id propagates to logs and response header; `/health` reachable without a token and reports `degraded` when a dependency mock fails; nothing constructs clients or fetches JWKS at import (test by importing the app module with network mocked to raise).
7. **Docs**: `docs/apim.md`, ≤ 25 lines: settings this API needs (`TENANT_ID`, `API_AUDIENCE`, `APIM_REQUIRE_HEADER`, `APIM_BACKEND_KEY` via Key Vault reference), what the APIM policy must do (validate-jwt or pass-through, headers it must forward, backend health probe path), and one line stating `RAG_URL`/`RAG_SCOPE` are configured on the web app, not here.
8. Add the settings to `.env.example` with placeholders only.
9. Commit: `Serve API behind APIM: token validation, correlation, health`. Phase report.

---

### PHASE 3 — in-process retrieval from the wiki index
**External contract (read-only):** Azure AI Search service `gmf-eastus2-devtest-ccoebot-search-service`, index `wiki-index`, written by the data-pipeline repo's wiki-ingest job. That job may drop and recreate the index at any time; "index not found" and zero hits are normal, non-fatal. Auth: `DefaultAzureCredential` only (managed identity in Azure, `az login` locally). No API keys.

**Discovery:**
1. Run `SearchIndexClient(endpoint, DefaultAzureCredential()).get_index("wiki-index")` and paste: all field names and types, which field is the key, which text field is searchable, the vector field name and dimensions, vector and semantic configuration names. From the dimensions, state which embedding deployment is implied (3072 → `text-embedding-3-large`; 1536 → `text-embedding-3-small`/`ada-002`). Write the discovered schema to `tests/fixtures/wiki_index_schema.json`. If the service is unreachable → **Access halt**.

**Build:**
2. `WikiRetriever` in the repo's existing package layout. Settings via the existing pattern: `SEARCH_ENDPOINT`, `WIKI_INDEX_NAME` (default `wiki-index`), `WIKI_EMBEDDING_DEPLOYMENT` (default = the deployment implied in discovery), `RETRIEVAL_TOP_K` (default 6). `SearchClient` and the embeddings client are constructed lazily on first use; nothing at import or startup.
3. `retrieve(query: str, top_k: int | None = None) -> list[Chunk]`: embed the query with `WIKI_EMBEDDING_DEPLOYMENT`; hybrid search = keyword on the discovered searchable text field + `VectorizedQuery` on the discovered vector field; use the semantic configuration if one exists, else skip. `Chunk` is a Pydantic model: `content`, `title`, `source_url`, `page_id`, `chunk_index`, `score` — mapped from the discovered field names, never guessed. Dedupe by `page_id` (best chunk per page, then fill remaining slots). Truncate total content to a configurable token budget (`RETRIEVAL_MAX_TOKENS`, default 4000, approximate by chars if no tokenizer is already in the repo).
4. Error policy: index-not-found, 429, 5xx, timeout, or auth error → WARNING log with the error class (never the query text at WARNING level), return `[]`, and expose `last_status` ∈ `ok | empty | unavailable`. Never raise into the agent loop.
5. Wire into the existing KB/fulfillment path as a pre-step: inject chunks as a delimited `Reference documents` block with `[n]` markers and a source list (`title — source_url`); instruct the agent to cite `[n]` and to say plainly when the references don't cover the question; when `last_status == unavailable`, tell the agent the wiki is temporarily unavailable so it doesn't guess. Keep the injection out of the guardrail gate's path.
6. Register one native Agent Framework function tool `search_wiki(query: str, top_k: int = 5) -> list[Chunk]` on the same agent, wrapping the same retriever, so it can re-query. Pydantic-validated signature. Do not register it through MCP.
7. **Tests** (mocked `SearchClient`/embeddings): the configured embedding deployment is used; hybrid query includes both keyword and vector parts; `page_id` dedupe; index-not-found and empty paths return `[]` with the correct `last_status`; context block formatting and citation numbering; nothing instantiated at import; **contract test** that loads `tests/fixtures/wiki_index_schema.json` and asserts the retriever's field mapping references only fields that exist in it.
8. If credentials allow, one live smoke test behind a marker (`@pytest.mark.live`, skipped by default) that runs a real query and asserts `last_status != "unavailable"`. Paste its output if you could run it; otherwise mark "not live-verified".
9. Add the settings to `.env.example` with placeholders only.
10. Commit: `Add in-process wiki index retrieval`. Phase report.

---

### Hard constraints (all phases)
- No edits to `azure-pipelines.yml`, `Dockerfile`, `infra/`, Terraform, Bicep, Key Vault, or any variable group.
- No `RAG_URL`, `RAG_SCOPE`, `CONNECTIONS__*`.
- No required-variable checks, startup assertions, or validation gates.
- No SharePoint code, settings, tools, or tests.
- Do not touch the guardrail gate, MCP client/gateway, or tool allowlist.
- New dependencies only if strictly required (`azure-search-documents`, a JWT/JWKS library), pinned, and named in the phase report.
- Small, reviewable diffs. One commit per phase. Never push. Never run a pipeline.
- Prerequisite outside your control (note it in the Phase 3 report if it fails): the App Service managed identity needs **Search Index Data Reader** on the search service.

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
