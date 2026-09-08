# Multi-Agent SWE Assistant

A multi-agent system that turns a natural-language software request into a
working, tested, documented Python project.

Final-year B.E. project. Built by two students under a zero-budget constraint:
every provider is free tier, every embedding runs locally.

---

## Pipeline

```
Requirement Analysis
        |
        v
    Planning  <----------------------+
        |                            |
        v                            |
    Research                         |
        |                            |
        v                            |
 Code Generation <---------+         |
        |                  |         |
        v                  |         |
     Review                | repair  |
        |                  | loop    |
        v                  |         |
  Test Writing             |         |
        |                  |         |
        v                  |         |
Sandboxed Execution -------+         |
        |  (tests pass)               |
        v                            |
  Documentation                      |
        |                            |
        v                            |
 Final Delivery ---------------------+
```

Ten conceptual stages, **eight implemented components**:

| Conceptual stage | How it is implemented |
|---|---|
| Requirement Analysis | agent |
| Planning | agent (absorbs Database Design) |
| Research | agent |
| Code Generation | agent (absorbs Database Design) |
| Review | agent |
| Test Writing | agent |
| Sandboxed Execution | agent |
| Debugging / Repair | **a loop edge back to Code Generation, not an agent** |
| Documentation | agent |
| Final Delivery | **a plain function (zip + git push), not an agent** |

---

## Architecture

| Layer | Choice |
|---|---|
| Orchestration | LangGraph `StateGraph`; nodes are agents, conditional edges route the repair loop, checkpointer enables human-in-the-loop pauses |
| Shared state | `app/models/state.py :: ProjectState` (Pydantic) — **frozen** |
| LLM access | `app/llm_client.py` — the single entry point for every model call |
| Backend | FastAPI, async, SSE for live run progress |
| Database | SQLAlchemy + Alembic; SQLite locally, Postgres via `DATABASE_URL` |
| RAG | ChromaDB + local `all-MiniLM-L6-v2` sentence-transformers |
| Frontend | React — New Run, Live Run, Artifacts, History |
| Sandbox | Docker SDK for Python, `--network=none`, memory cap, hard timeout |

### How the agents communicate

Agents **never call each other**. Each one reads `ProjectState`, does its work,
and returns a *partial* dict that LangGraph merges back into the state. This is
what lets the two of us work on different agents at the same time without our
changes colliding.

```
        +---------------- ProjectState ----------------+
        |                                              |
   read |                                              | merge partial
        v                                              |
    [ Agent ] -- returns dict, e.g. {"plan": [...]} ---+
```

---

## Repository layout and ownership

Ownership is by directory, so the two of us never edit the same file.

| Path | Owner | Purpose |
|---|---|---|
| `app/agents/` | Member A | One module per agent |
| `app/orchestrator/` | Member A | LangGraph graph, edges, checkpointer |
| `app/rag/` | Member A | ChromaDB indexing and retrieval |
| `prompts/` | Member A | Prompt templates, versioned |
| `eval/` | Member A | Benchmark tasks and scoring |
| `app/api/` | Member B | FastAPI routes, SSE stream |
| `app/db/` | Member B | SQLAlchemy models, Alembic migrations |
| `app/sandbox/` | Member B | Docker execution harness |
| `frontend/` | Member B | React app |
| Docker, CI | Member B | Dockerfiles, GitHub Actions |
| `app/models/` | **shared, frozen** | `ProjectState` and friends — changes need both of us to agree |
| `app/llm_client.py` | **shared** | Single LLM gateway |
| `config/` | shared | `providers.yaml` routing table |
| `scripts/` | shared | `verify_setup.py` |
| `tests/` | shared | Mirrors the package layout |
| `docs/` | shared | ADRs |

---

## Getting started

```bash
git clone https://github.com/krithickvivek/multi-agent-swe-assistant.git
cd multi-agent-swe-assistant

python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS / Linux:
source .venv/bin/activate

pip install -r requirements.txt

cp .env.example .env      # then paste your free-tier keys into .env
```

Prove the environment works on your laptop:

```bash
python scripts/verify_setup.py
```

It prints PASS / FAIL / SKIP for each provider key, Docker, SQLite, ChromaDB,
and every import in `requirements.txt`. A provider you have not signed up for
yet is reported SKIP, not FAIL.

Run the tests (no network required):

```bash
pytest
```

---

## Hard constraints

These are non-negotiable and are enforced in code and in review:

1. **Zero budget.** Free-tier providers only. Embeddings run locally.
2. **Generated code never runs on the host.** Docker only, `--network=none`, memory cap, hard timeout.
3. **All LLM calls go through `app/llm_client.py`.** No provider SDK is even installed.
4. **Every LLM output is validated into a Pydantic model.** No free text between agents.
5. **Agents never call other agents.** Read state, return a partial update.
6. **Secrets from environment variables only.**
7. **Tests run without network access,** against mocked responses.

## Contributing

Conventional commits, small PRs, no commits directly to `main`.

## License

MIT — see [LICENSE](LICENSE).
