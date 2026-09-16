Context: core-ccoe-ccoebot-api is Cleo's API (Python, Microsoft Agent Framework, Azure App Service). Branch feature/apim-backend, based on 0b18151. Architecture: web app → Azure API Management → this API. This API is the BACKEND behind APIM; it never calls APIM. RAG_URL / RAG_SCOPE are web-app settings. Pre-existing references to RAG_URL, RAG_SCOPE, and CONNECTIONS__SERVICE_CONNECTION* in this repo predate 0b18151 and must not be added to, removed, renamed, or refactored.

Goal: make the APIM integration work, driven by real Azure DevOps pipeline runs. I will run the pipeline and paste the failing log into this chat. You diagnose and fix. I re-run. Repeat until green.

Your loop, every time I paste a log:
1. DIAGNOSE — quote the first real error (not the cascade after it), name the stage/job/step it came from, and state in one sentence the root cause and whether it is (a) code in this repo, (b) pipeline YAML, (c) App Service / APIM / Entra configuration, or (d) an environment issue (feed, agent pool, permissions). If you cannot tell from the log, ask me for the specific extra output you need — do not guess.
2. FIX — apply the smallest change that addresses that root cause only. Category (a): edit code. Category (b): edit azure-pipelines.yml only for the failing step; show me `git diff -- azure-pipelines.yml` before committing. Category (c): do NOT edit anything; write the exact setting/policy/role that must be changed, where, and by whom, so I can do it in the portal or ask the owner. Category (d): same — tell me what to fix outside the repo.
3. VERIFY LOCALLY where possible — import the app with network mocked to raise; run only the tests you added, synchronous pytest, `pytest tests/<your files> -q -o addopts=""`. Do not install pytest-asyncio, do not change test config, do not run the full suite. Paste output. If nothing can be verified locally, say "pipeline-verified only".
4. COMMIT — one commit per fix, message "APIM: <what and why>". Never push, never queue a pipeline; I do both.
5. REPORT — 4 lines: error / root cause / change made / what I should check in the next run.

Hard rules:
- Never weaken, skip, or delete a test, health check, or token validation to get green.
- Never add required-variable gates, startup assertions, or `az` calls to read variable groups in the pipeline.
- Never touch Dockerfile, infra/, Terraform/Bicep, Key Vault, or any variable group; if a value must live there, name it and stop.
- Never put a secret, token, key, tenant id, or URL in code or .env.example; placeholders only.
- Do not touch the guardrail gate, MCP client/gateway, tool allowlist, or any search/retrieval code.
- If the same error survives two fixes, stop and give me your two best hypotheses with the evidence for each.

Before the first log arrives: run `git status`, `git log --oneline 0b18151..HEAD`, and `git diff 0b18151 --stat`, paste them, and confirm you're on a clean feature/apim-backend. Then wait for my log.

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
