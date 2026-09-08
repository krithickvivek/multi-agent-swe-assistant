# Project: Multi-Agent AI Software Engineering Assistant

## What this is
A final-year B.E. project. A multi-agent system that takes a natural-language software
request and produces a working, tested, documented Python project. Pipeline:
Requirement Analysis → Planning → Research → Code Generation → Review → Test Writing →
Sandboxed Execution → Repair Loop → Documentation → Delivery.

## Who is building it
Two final-year students. Beginners in Agentic AI, comfortable with Python basics.
60-day deadline, three college reviews, and a research paper at the end.
- Member A owns: app/agents/, app/orchestrator/, prompts/, app/rag/, eval/
- Member B owns: app/api/, app/db/, app/sandbox/, frontend/, Docker, CI

## Hard constraints — never violate these
1. ZERO BUDGET. Free-tier LLM providers only (Gemini, Mistral, Groq, OpenRouter).
   Never add a dependency that requires payment. Embeddings run locally via
   sentence-transformers — never call a paid embedding API.
2. Generated code NEVER executes on the host. Always inside Docker with
   --network=none, a memory cap, and a hard timeout. No subprocess on the host, ever.
3. Every LLM call goes through app/llm_client.py. No direct provider SDK calls anywhere else.
4. Every LLM output is parsed into a Pydantic model. No free-text passing between agents.
5. Agents never call other agents. They read ProjectState and return a partial update.
6. Secrets come from environment variables only. Never hard-code a key or token.
7. Tests must run without network access, using mocked LLM responses.

## Architecture
- Orchestration: LangGraph StateGraph. Nodes = agents. Conditional edges for
  routing and the repair loop. Checkpointer for human-in-the-loop pauses.
- Shared state: app/models/state.py :: ProjectState (Pydantic). This is FROZEN.
  If a change is genuinely needed, stop and ask before editing it.
- Backend: FastAPI, async, SSE for live run progress.
- DB: SQLAlchemy + Alembic. SQLite locally, Postgres via DATABASE_URL.
- RAG: ChromaDB + local sentence-transformers (all-MiniLM-L6-v2).
- Frontend: React. Four screens: New Run, Live Run, Artifacts, History.
- Sandbox: Docker SDK for Python.

## Model routing (config/providers.yaml — all OpenAI-compatible endpoints)
| Role         | Provider | Model                    |
|--------------|----------|--------------------------|
| requirement  | mistral  | mistral-small-latest     |
| planner      | gemini   | gemini-2.5-flash         |
| researcher   | gemini   | gemini-2.5-flash         |
| codegen      | mistral  | codestral-latest         |
| reviewer     | groq     | llama-3.3-70b-versatile  |
| testwriter   | mistral  | codestral-latest         |
| repair       | gemini   | gemini-2.5-flash         |
| docs         | groq     | llama-3.3-70b-versatile  |
Every role has a fallback chain. Rate limits are real — token-bucket limiting,
exponential backoff on 429, automatic failover, and response caching are mandatory.

## Scope discipline
The 10 conceptual stages map to 8 implemented components:
- Debugging is a LOOP EDGE back to Code Generation, not an agent.
- Final Delivery is a plain function (zip + git push), not an agent.
- Database Design folds into Planning and Code Generation.
Do not propose features outside the current phase. If you think something is
missing, say so in one sentence and continue with what was asked.

## How I want you to work with me
- I am a beginner in Agentic AI. Explain WHY before HOW, in plain language.
- Ask clarifying questions before writing code if the request is ambiguous.
- One component per session. Do not refactor files I did not ask about.
- Write the test alongside the code, not afterwards.
- After each change, tell me in 2-3 lines what changed and how to verify it,
  because I have to defend every line of this in a viva.
- If I ask for something that contradicts a hard constraint above, refuse and
  explain why. Do not silently work around it.
- Conventional commits. Small PRs. No commits directly to main.
