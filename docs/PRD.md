# PRD — Lenny Growth Assistant

## 1. User

Early-stage PM / founder / growth generalist who reads growth content (a la
Lenny's Newsletter) but doesn't have a dedicated growth team to turn that
knowledge into decisions. Secondary user: an FDE evaluator who needs to
verify the product works end-to-end and judge my technical/product choices.

## 2. Problem

Growth knowledge is scattered across long-form articles and frameworks.
When someone has a concrete question — "is my LTV:CAC healthy?", "what's a
growth loop, and does it apply to my product?" — they either re-read a
newsletter archive or guess. There's no single place that (a) answers from
trusted growth frameworks instead of generic LLM knowledge, and (b) does
the actual math when the question is quantitative.

## 3. Success Metric

Primary: % of user questions answered with a grounded source citation
(target ≥ 70% for questions that match the corpus) vs. an ungrounded
fallback — because the core value proposition is "grounded, not generic."
Secondary (would need real usage to measure): session length / return-usage
rate as a proxy for whether answers were actually useful.

## 4. Assumptions

- A small, curated corpus (5 documents) is enough to demonstrate the
  retrieval + grounding pattern; production would need dozens–hundreds of
  ingested articles.
- Users mostly ask two kinds of questions: conceptual ("what's a growth
  loop") and quantitative ("is my CAC/LTV healthy") — hence two dedicated
  skills instead of one general-purpose chat.
- A local model (Ollama) is an acceptable default for development/demo so
  the app has zero required paid dependencies; cloud (Anthropic) is an
  opt-in upgrade for quality.

## 5. Scope

**In scope (v1):**
- Chat UI with conversation persistence
- Retrieval-grounded answers over a local markdown corpus
- Deterministic CAC/LTV calculator skill
- Model provider toggle (local Ollama / cloud Anthropic)
- Source/artifact viewer showing which document(s) grounded an answer

**Out of scope (v1, noted as future work in architecture.md):**
- User accounts / auth / multi-tenant data isolation
- Ingesting arbitrary user-uploaded documents into the corpus
- Streaming token-by-token responses
- A vector database (pgvector) — v1 uses in-process TF-IDF, sufficient at
  this corpus size

## 6. Flows

**Flow A — Grounded question:**
User asks a conceptual growth question → router matches no calculator
keywords → retriever finds relevant corpus chunk(s) → model answers using
that context → UI shows the answer plus a Sources panel with the matched
documents and relevance scores.

**Flow B — Calculation question:**
User asks a CAC/LTV question with numbers → router detects calculator
keywords → calculator agent parses numbers and computes the ratio in
Python (not the LLM) → model is asked only to phrase the pre-computed
result in a sentence → UI shows the answer with no source panel (nothing
to cite — it's arithmetic).

**Flow C — Ungrounded fallback:**
User asks something outside the corpus and not a calculation → general
agent answers from model knowledge → UI shows the answer tagged "general"
with no Sources panel, signaling to the user that this wasn't
source-grounded.

## 7. Acceptance Criteria

- [ ] A user can send a message and receive a reply persisted to Postgres
- [ ] A conceptual question about growth loops/aha moments/retention/
      prioritization returns an answer plus ≥1 source with a title and
      snippet
- [ ] A CAC/LTV question with two numbers returns the correct ratio and a
      healthy/unhealthy judgment (ratio ≥ 3 = healthy)
- [ ] Switching `MODEL_PROVIDER` between `ollama` and `anthropic` changes
      which backend answers, with no code changes required
- [ ] Reloading a conversation via `GET /api/conversations/{id}` returns
      the full message history in order
- [ ] The UI clearly distinguishes grounded vs. ungrounded answers

## 8. Risks

- **Small corpus → weak retrieval on edge-case questions.** Mitigated by
  the general-agent fallback so users always get *an* answer, clearly
  labeled as ungrounded.
- **Keyword-based routing misclassifies.** E.g., "what's a good CAC
  benchmark" (conceptual) could be misrouted to the calculator, which will
  correctly report "needs_clarification" rather than hallucinate numbers —
  a deliberate fail-safe, not a crash.
- **Local model (Ollama) quality/latency varies by hardware.** The toggle
  to Anthropic exists specifically so evaluators without a fast local GPU
  can still assess answer quality.

## 9. Implementation Plan

1. Data layer: Postgres schema + SQLAlchemy models (conversations, messages)
2. Retrieval: TF-IDF corpus + retriever
3. Agents: calculator, retrieval, general + router
4. Model client abstraction (Ollama/Anthropic)
5. FastAPI routes: chat, conversations, health
6. Frontend chat UI with source viewer
7. Tests: unit (router, retriever, calculator) + integration (API, persistence)
8. Docs: README, design.md, architecture.md
9. Deploy (Railway/Supabase) + record demo Video 
