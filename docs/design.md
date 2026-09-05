# Design — Lenny Growth Assistant

## UI/UX Principles

1. **Trust through transparency.** Every answer is visibly tagged with
   which skill produced it (`retrieval` / `calculator` / `general`) and,
   for retrieval, exactly which source documents were used.
2. **No dead ends.** The general-agent fallback means there's always a
   reply, even for out-of-corpus questions — just honestly labeled as
   ungrounded rather than silently failing.
3. **Minimal chrome.** A single chat column plus a collapsible sources
   panel — no navigation, settings screens, or onboarding flow needed.

## Information Architecture 

The provider badge calls `/api/health` on load, so it's instantly visible
whether the backend is running local Ollama or cloud Anthropic.

## Key Interaction States

- Empty state: pre-seeded assistant message explaining what it can do.
- Loading: send button disables while a request is in flight.
- Grounded answer: tagged `retrieval`; Sources panel shows title, snippet, score.
- Calculation answer: tagged `calculator`; no sources panel (arithmetic isn't "sourced").
- Ungrounded answer: tagged `general`; no sources panel — the absence is the signal.
- Error state: a distinct red-tinted inline bubble, not a silent console error or alert.

## Responsive Behavior

Above 720px: chat + 300px sources panel side by side. Below 720px: sources
panel hides entirely rather than cramming a small screen.

## Accessibility Considerations

Semantic `<input>` and `<button>` elements, Enter-to-send, WCAG-AA contrast
in the color palette. Known gap: no `aria-live` region yet for
screen-reader announcement of new messages — documented as a next step.

## Key Design Decisions

- Every message is tagged with its producing agent, always — the single
  most important trust signal for an AI product.
- No streaming in v1 — latency is low enough at this corpus/model size
  that streaming would add complexity without a proportional UX win.
- Single HTML/JS file for the frontend, no framework build step — keeps
  clone-and-run friction near zero for an evaluator.
