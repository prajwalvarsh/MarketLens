# CLAUDE.md — MarketLens

This file is read automatically at the start of every Claude Code
session, after every /compact, and after every fresh chat. It is
the project constitution. All other spec files are referenced from
here — read them before building anything.

---

## Project

MarketLens is a multi-agent Indian equity (NSE/BSE) analysis system.
Four specialist agents run in parallel via LangGraph Send(), analysing
technicals, news sentiment, sector rotation, and derivatives data.
An AutoGen critic conversation cross-checks their outputs and scores
confidence. A report writer produces a structured, plain-English
analysis report.

This is NOT a trading bot. It produces research reports. A human
reads the report and decides what to do. The agent recommends;
the human acts.

---

## Tech Stack

- Language: Python 3.11
- Package manager: uv (never use pip directly)
- Linter/formatter: ruff
- Testing: pytest + pytest-cov
- LLM orchestration: LangGraph
- Multi-agent critique: AutoGen (pyautogen)
- LLM API: DeepSeek V4 Flash via OpenRouter (OpenAI-compatible) — the only
  inference backend. No self-hosted model in this project — see dev.md
  "Key Decisions" for why vLLM was deliberately excluded.
- API framework: FastAPI
- UI: single static HTML page (vanilla JS, calls the API directly) —
  no frontend framework. See product.md for rationale.
- Tracing: per-request span tree, written to `eval_results/traces/`
  and viewable via FastAPI route. See dev.md "Tracing" section.
- Container: Docker
- Orchestration: Kubernetes (minikube locally, GKE in cloud)
- CI/CD: GitHub Actions
- Delivery surfaces: MCP server (exposes MarketLens as an `analyse_stock`
  tool for MCP-compatible AI clients) + Telegram (optional report
  delivery via `notify="telegram"`, routed through a Telegram MCP
  server / Bot API integration). Both are thin layers on top of the
  existing `/analyse` endpoint — no agent logic is duplicated. See
  dev.md "Delivery Surfaces" section.

---

## Spec Files

Read all of these before starting any sprint:

- Product spec: @specs/product.md — what we're building, why, acceptance criteria
- Dev spec: @specs/dev.md — architecture, agent design, state schema, repo structure
- Test spec: @specs/test.md — test strategy, all test cases, coverage, evals, ruff
- Deploy spec: @specs/deploy.md — Docker, Kubernetes, CI/CD, cost

---

## Workflow Rules

- Read all spec files at the start of every session
- After each sprint, update specs before starting the next (see sprint review prompt)
- Never add a feature not in product.md without asking first
- Commit after every completed task
- Each sprint task gets clean context — don't carry unrelated history forward
- Stop and ask if a spec section is ambiguous, never assume
- Sprints are intentionally short (one deliverable each) — do not bundle
  multiple sprints' work into one session even if it seems efficient

---

## Architecture Rules

1. Agents live in `agents/`. One file per agent. One node function
   per file. No business logic — only orchestration.
2. Tools live in `tools/`. Pure functions. No LLM calls inside tools.
   No side effects beyond returning a value.
3. All LLM calls go through `config.get_llm_client()` only.
   Never instantiate an OpenAI client directly in agent files.
4. `AgentState` is defined in `state.py`. Never add fields anywhere else.
5. The graph is defined only in `graph.py`. No graph logic in agents.
6. FastAPI routes live in `api/`. No business logic in routes.
7. Environment variables are loaded only in `config.py`.
8. Every agent node must call `log_agent_run()` from `observability/`
   AND emit a trace span via `observability/tracing.py`.
9. Eval fixtures live in `tests/fixtures/`. Never hardcode test data
   inline in eval test files.
10. There is no local/self-hosted LLM backend in this project. Do not
    add vLLM, Ollama, or any GPU-dependent inference path unless the
    spec is explicitly updated first.
11. `delivery/` and `mcp_server/` must never import from `agents/`,
    `graph/`, or `state.py`. They consume only the finished `Report`
    schema returned by the API. A delivery failure must never cause
    `/analyse` to return a non-200 response for an otherwise
    successful analysis.

---

## Conventions

- File naming: snake_case for Python, kebab-case for YAML/config
- Folder structure: domain-first (`agents/`, `tools/`, `api/`), not type-first
- Testing: pytest for unit/integration, LLM-as-judge for evals
- Commit messages: conventional commits (`feat:`, `fix:`, `test:`, `chore:`)
- Docstrings: every public function in `tools/` and `agents/` gets one

---

## Commands

### Install dependencies
```
uv sync
```

### Lint and format
```
uv run ruff check .
uv run ruff check . --fix
uv run ruff format .
```

### Unit tests (fast, no network, no LLM)
```
uv run pytest tests/unit -v
```

### Integration tests (mocked LLM + mocked data)
```
USE_MOCK_LLM=true USE_MOCK_DATA=true uv run pytest tests/integration -v
```

### Full test suite with coverage (what CI runs)
```
uv run pytest tests/unit tests/integration \
  --cov=agents --cov=tools --cov=graph --cov=api --cov=observability \
  --cov-report=term-missing \
  --cov-report=html:coverage_html \
  --cov-fail-under=80 \
  -v
```

### Eval suite (hits live LLM — costs tokens, run manually only)
```
uv run pytest tests/evals/ -v -s \
  --eval-report=eval_results/full_$(date +%Y%m%d).json
```

### View a request trace
```
curl http://localhost:8080/trace/<run_id>
```

### Run the MCP server locally (Sprint 8)
```
cd mcp_server
uv run server.py
```

### Test Telegram delivery (Sprint 8, requires TELEGRAM_BOT_TOKEN)
```
curl -X POST http://localhost:8080/analyse \
  -H "Content-Type: application/json" \
  -d '{"symbol": "RELIANCE", "period": "3mo", "notify": "telegram", "chat_id": "<your_chat_id>"}'
```

### Local dev
```
docker compose up
```

### Kubernetes (local)
```
minikube start --driver=docker
minikube addons enable ingress
kubectl apply -f k8s/
kubectl get pods -w
minikube tunnel
```

---

## Environment Variables

Copy `.env.example` to `.env` and fill in:

```
LLM_BASE_URL=https://openrouter.ai/api/v1
LLM_MODEL=deepseek/deepseek-v4-flash
OPENROUTER_API_KEY=your-key-here
NEWSDATA_API_KEY=your-key-here
USE_MOCK_LLM=false
USE_MOCK_DATA=false
EVAL_JUDGE_MODEL=deepseek/deepseek-v4-flash
EVAL_SCORE_THRESHOLD=3.5
TRACE_OUTPUT_DIR=eval_results/traces
TELEGRAM_BOT_TOKEN=your-bot-token-here
```

`mcp_server/` reads its own small `.env` (separate from the main app):

```
AGENT_API_BASE_URL=http://localhost:8080
```

---

## Current Sprint

Sprint 0 — specs complete, no code written yet.

Update this section after every sprint review. See product.md for the
full 8-sprint breakdown.

---

## Adding a New Agent

1. Create `agents/your_agent.py` — single node function
2. Add tools to `tools/your_tools.py` — pure functions only
3. Add output field to `AgentState` in `state.py`
4. Register node in `graph.py`
5. Add `Send()` call in `agents/orchestrator.py`
6. Add unit tests in `tests/unit/test_your_tools.py`
7. Add eval cases to `tests/fixtures/eval_cases.json`
8. Update `tests/integration/test_graph.py` mock fixtures
9. Ensure the node emits both a structured log entry and a trace span
10. Run full test suite — must pass at 80% coverage

---

## Adding a New Delivery Channel

1. Create `delivery/your_channel.py` — formats `Report` + sends
2. Add a formatter function to `delivery/formatters.py` if shared
   formatting logic is needed across channels
3. Extend the `notify` enum in `api/models.py` to include the new value
4. Add unit tests for message formatting in `tests/unit/`
5. Add integration tests for success + failure paths in `tests/integration/`,
   mocking the underlying send call
6. Confirm the new channel's failure cannot cause `/analyse` to return
   a non-200 — this is a hard architecture rule, not optional
7. Do not import anything from `agents/`, `graph/`, or `state.py`
   into the new delivery module