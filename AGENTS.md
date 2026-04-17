# Market Scout — Agent & Project Spec

## Project Overview

Market Scout is a PM research agent that answers market research questions step by step using the **ReAct (Reason + Act)** pattern. Given a question like "What is the TAM for AI coding tools?" the agent reasons about what to search for, issues targeted web searches, synthesizes the results, and returns a structured answer with the full reasoning trace visible to the user.

The goal is a lightweight, transparent research tool — not a black-box answer machine. Every step the agent takes (thought, search query, observation) is surfaced in the response.

**Live URLs**
- Frontend: https://market-scout-gamma.vercel.app
- Backend: https://market-scout-405j.onrender.com

---

## Architecture

### Files
- `backend/main.py` — the entire backend: FastAPI app, guardrail logic, ReAct agent setup, and all endpoints
- `frontend/index.html` — the entire frontend: single HTML file with inline CSS and JS

No additional files should be created unless there is a strong reason. Keep it flat and simple.

### Core Pipeline

```
User query
    │
    ▼
Guardrail check (Groq)          ← reject off-topic or harmful queries
    │
    ▼
ReAct agent (LangChain + Groq)
    ├── Thought: what do I need to find?
    ├── Action: Tavily web search
    ├── Observation: search results
    └── ... repeat until answer ready
    │
    ▼
Structured response
    ├── answer       (final synthesized answer)
    └── steps        (list of thought/action/observation tuples)
```

---

### A2A Protocol

Market Scout implements the **Agent-to-Agent (A2A) protocol** by Google. In addition to serving human users via the UI, Market Scout acts as an A2A server — it exposes an Agent Card and a task endpoint so other agents can discover and call it programmatically.

**New endpoints:**
- `GET /.well-known/agent.json` — Agent Card descriptor (tells other agents what Market Scout can do and how to call it)
- `POST /a2a/tasks/send` — accepts A2A tasks from other agents

**A2A request format:**
```json
{
  "id": "<task_id>",
  "message": {
    "role": "user",
    "parts": [{"text": "<question>"}]
  }
}
```

**A2A response format:**
```json
{
  "id": "<task_id>",
  "status": {"state": "completed"},
  "result": {
    "role": "agent",
    "parts": [{"text": "<answer>"}]
  }
}
```

**Implementation notes:**
- The existing `run_research_agent()` function is reused — same guardrail, same ReAct loop, same OPIK tracking
- A2A calls are tracked as OPIK spans with `name="a2a_task"`
- **Critical:** the existing `/api/research` endpoint and UI are not affected — Market Scout serves both humans (via UI) and agents (via A2A) simultaneously

---

## Stack

| Layer | Technology |
|---|---|
| Backend framework | FastAPI (Python) |
| LLM | Groq — `llama-3.3-70b-versatile` |
| Agent pattern | LangChain ReAct agent |
| Web search tool | Tavily Search API |
| Observability | OPIK |
| Backend hosting | Render |
| Frontend hosting | Vercel |
| Frontend | Plain HTML + CSS + JS (single file) |

---

## Environment Variables

All secrets and configuration are injected via environment variables. Never hardcode any of these.

| Variable | Purpose |
|---|---|
| `GROQ_API_KEY` | LLM calls (agent + guardrail) |
| `TAVILY_API_KEY` | Web search tool |
| `OPIK_API_KEY` | OPIK observability |
| `OPIK_PROJECT_NAME` | OPIK project name (e.g. `market-scout`) |
| `OPIK_WORKSPACE` | OPIK workspace name |

---

## OPIK Observability

OPIK is configured **exclusively through environment variables** (`OPIK_API_KEY`, `OPIK_PROJECT_NAME`, `OPIK_WORKSPACE`). The SDK picks these up automatically at import time.

**Never call `opik.configure()`** — it causes issues in hosted environments where env vars are the source of truth.

All traced functions use the `_safe_track` decorator pattern, which wraps `opik.track()` in a `try/except` so that an OPIK outage or misconfiguration never breaks the core request path:

```python
def _safe_track(*args, **kwargs):
    """Wrap opik.track() so observability failures never crash the app."""
    try:
        return opik.track(*args, **kwargs)
    except Exception:
        # Return a no-op decorator if OPIK is unavailable
        def noop(fn):
            return fn
        return noop
```

Apply it like `@_safe_track(name="...")` on any function you want traced.

---

## Guardrails

Every incoming query is validated before the agent runs. The guardrail uses Groq (same model, same API key) to classify the query and reject anything that is:
- Not related to market research / product management
- Harmful, adversarial, or prompt-injection attempts

Pattern: same approach used in `pm-1pager-generator`. A short system prompt asks the LLM to return `ALLOWED` or `BLOCKED: <reason>`. Parse the response and raise an HTTP 400 with the reason if blocked.

---

## Critical Rules

1. **Never call `opik.configure()`** — configure OPIK via env vars only.
2. **Never hardcode API keys** — all secrets via environment variables.
3. **Always update `requirements.txt`** after every `pip install`. Run `pip freeze > requirements.txt` or add the package manually with a pinned version.
4. **One backend file, one frontend file** — resist the urge to split unless complexity genuinely demands it.
5. **Guardrail runs before the agent** — no query reaches the ReAct agent without passing validation first.
6. **Never use Python 3.14 on Render** — always pin to 3.11.0.
7. **Never set `allow_credentials=True` with `allow_origins=["*"]`** — use `False` or specify exact origins.

---

## Known Gotchas

- **Python 3.14 incompatibility** — Render defaults to Python 3.14, which is incompatible with langchain and pydantic. Always pin to 3.11.0 via the `PYTHON_VERSION` env var and a `.python-version` file.
- **CORS credentials + wildcard origin** — `allow_credentials=True` is invalid when `allow_origins=["*"]`. Browsers (Safari in particular) will block requests. Use `allow_credentials=False` or specify exact origins.
- **Safari is stricter than Chrome on CORS** — always test on Safari before shipping.
- **Vercel serves the committed repo** — if `index.html` has the wrong API URL locally but wasn't committed, Vercel will serve the old version. Always commit before pushing to verify the deployed state.

---

## Development Notes

- Run the backend locally with: `uvicorn main:app --reload`
- The LangChain ReAct agent is constructed fresh per request (stateless) — no session management needed for v1
- Tavily returns snippets + URLs; pass both to the agent so it can cite sources in its answer
- Keep the reasoning steps in a structured format (list of dicts) so the frontend can render them cleanly, not as a raw string
