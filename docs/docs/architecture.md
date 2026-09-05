# Architecture — Lenny Growth Assistant

## Database Schema

conversations: id (PK), created_at. messages: id (PK), conversation_id (FK to conversations, cascade delete), role ("user"|"assistant"), content, agent_used ("retrieval"|"calculator"|"general"), sources_json (JSON list of {title, snippet, score}), created_at. One conversation has many messages. agent_used and sources_json are stored per-message so the answer's provenance is auditable after the fact without re-running anything.

## API Endpoints

GET /api/health — liveness + which model provider is active. POST /api/chat — send a message, get a routed reply (request: {conversation_id, message}; response: {conversation_id, reply, agent_used, sources}). GET /api/conversations/{id} — full message history for a conversation. Omitting conversation_id in POST /api/chat starts a new conversation.

## Component Boundaries

frontend/ is a static chat UI with no build step. app/main.py wires the FastAPI app and startup. app/config.py is the single source of truth for env-driven settings. app/routes/ is the HTTP layer only (validation, DB session, calling the router). app/agents/router.py holds the routing decision and prompt assembly. app/agents/*_agent.py holds individual skills (calculator today, extensible). app/agents/model_client.py is the provider-agnostic LLM call. app/retrieval/ holds the corpus and TF-IDF retriever. app/models/db.py holds SQLAlchemy models and session management. tests/ holds unit tests (router/retriever/calculator) and integration tests (API + persistence, via SQLite). Each layer only talks to the layer directly below it — routes never touch the retriever or model client directly, only router.route() — which keeps routing/prompting logic unit-testable without spinning up FastAPI or a database.

## Ingestion / Retrieval Flow

At startup, corpus/*.md files are fit into a TfidfVectorizer in-process. On each user query, the query is vectorized and compared via cosine similarity against the corpus to get top-k chunks, which are injected into the LLM prompt as "SOURCE CONTEXT" and also returned to the UI as structured {title, snippet, score} objects. No vector database is used at this corpus size (5 documents) since in-process TF-IDF is simpler to run locally and sufficient for correctness at this scale — this, along with the model toggle below, is the "one important technical trade-off" worth mentioning in the demo. Scaling path: swap TfidfVectorizer for embeddings (sentence-transformers locally, or an embeddings API) stored in Postgres via the pgvector extension (Supabase supports this natively), replacing the cosine-similarity call with a SQL ORDER BY embedding distance query — the router/agent code wouldn't need to change, only the Retriever would.

## Agent Routing

Keyword/heuristic routing in app/agents/router.py, in priority order: (1) calculator — triggered by growth-metric keywords (cac, ltv, payback, churn); deterministic Python math, the LLM only phrases the result; (2) retrieval — triggered when the corpus has any positive-similarity match; the LLM answers grounded in the retrieved context; (3) general — fallback; the LLM answers from its own knowledge and the response is tagged so the UI never presents it as sourced. Trade-off: an LLM-based router would generalize better to phrasing the keyword approach misses, at the cost of an extra model call per message (latency + cost) and non-deterministic routing that's harder to reproduce and unit test. Given the small, well-defined skill set in v1, deterministic keyword routing was chosen for speed, zero added latency, zero added cost, and 100%-reproducible unit tests.

## Model Toggle

app/agents/model_client.py defines one interface, complete(system, messages) -> str, with two implementations: OllamaClient (calls a local Ollama server's /api/chat, no API key) and AnthropicClient (calls the Anthropic Messages API). Selected via the MODEL_PROVIDER env var; nothing else in the codebase branches on provider, so the same code runs fully offline against Ollama or upgraded to Anthropic just by changing an env var.

## Security

No secrets are committed: .env is git-ignored and .env.example documents required variables with empty placeholder values. API keys are only ever read from environment variables, never hardcoded or logged. CORS is wide open for demo/evaluation convenience; a production deployment should restrict this to the actual frontend origin. There is no auth/multi-tenancy in v1 — every visitor shares the same conversation space unless the frontend is extended to pass a user/session identifier. All SQL access goes through SQLAlchemy's ORM/parameterized queries — no raw string-interpolated SQL anywhere in the codebase.

## Deployment Topology

The browser loads static HTML/JS served by FastAPI's StaticFiles. The FastAPI app runs on a host like Railway. It talks to PostgreSQL (hosted on Supabase) and to either a local Ollama instance (dev) or the Anthropic API (cloud/prod). Local dev runs uvicorn with --reload, Postgres via Docker or a local install, and Ollama on the same machine. Cloud deploy has Railway hosting the FastAPI app, Supabase hosting Postgres, and MODEL_PROVIDER set to anthropic since a deployed container has no local Ollama to reach. For observability, /api/health exposes provider and liveness for uptime checks, and FastAPI's default request logging covers basic access logs; structured logging and error tracking (e.g. Sentry) are noted as a follow-up gap.
