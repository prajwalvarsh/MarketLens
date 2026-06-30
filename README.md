# MarketLens

**A multi-agent research system for Indian equities.** Four specialist
AI agents analyse technicals, news sentiment, sector rotation, and
derivatives positioning in parallel — then argue with each other
before producing a report.

> This is a research tool, not a trading bot. It recommends; a human acts.

---

## The Problem

Retail investors in India — 100M+ added post-COVID — have access to
raw price charts and headlines but no synthesised, multi-signal
analysis in plain language. Wealth managers provide that synthesis
manually for HNI clients. Existing retail apps (Zerodha, Groww) show
data; none of them reason about it.

MarketLens does the synthesis: given a stock symbol, it returns a
single report with a signal, a confidence score, and an explanation —
including when its own signals disagree with each other.

---

## Architecture

```
                          POST /analyse { symbol }
                                  ↓
                          [ orchestrate ]
                                  ↓
              ┌───────────┬──────────────┬───────────────┐
              ↓           ↓              ↓                ↓
        [ technical ] [ news ]     [ sector ]      [ derivatives ]
         RSI, MACD     sentiment    FII/DII flow    PCR, max pain
         patterns      key events   rel. strength   IV
              │           │              │                │
              └───────────┴──────────────┴───────────────┘
                                  ↓
                            [ reduce ]
                                  ↓
                   [ critique ]  ←─── AutoGen conversation
                   recommender ↔ critic, max 3 rounds
                                  ↓
                  confidence < 0.6? → retry (max 2x)
                                  ↓
                          [ write_report ]
                                  ↓
                       structured JSON report
```

The four specialist agents run **genuinely in parallel** via LangGraph's
`Send()` fan-out — not just dispatched-and-awaited sequentially. Every
request is fully traced; `GET /trace/{run_id}` returns the span tree
for that run, and the overlapping timestamps on the four specialist
spans are the proof, not just a claim in this README.

Full architecture rationale, including every "why X instead of Y"
decision, lives in [`specs/dev.md`](specs/dev.md).

---

## What Makes This Technically Interesting

**Genuine multi-agent parallelism, not a pipeline wearing a multi-agent
costume.** Each specialist agent has its own focused context window and
its own tool set — the technical agent never sees a single news headline.
[`specs/dev.md`](specs/dev.md) explains why that separation matters for
reasoning quality, not just for show.

**A critic that argues, not just scores.** The critique step is an
AutoGen conversation — a recommender agent proposes a signal, a critic
agent challenges it, and they go back and forth (max 3 rounds) before
a confidence score is settled. Conflicting signals between domains are
explicitly surfaced in the final report, not silently averaged away.

**An eval suite with a deliberately independent judge.** Faithfulness
and reasoning quality are scored by a different model family than the
one generating the reports — DeepSeek V4 Flash generates, a separate
model judges — to avoid the self-preference bias models show when
evaluating their own output. See [`specs/test.md`](specs/test.md).

**A coverage gate that's enforced, not aspirational.** 80% minimum on
`agents/`, `tools/`, `graph/`, `api/`, `observability/` — CI fails the
build below that threshold. Ruff runs first and fails fast on lint.

**Self-contained request tracing with zero external dependencies.** No
LangSmith, no Jaeger — a lightweight span-tree tracer writes one JSON
file per request, retrievable via `GET /trace/{run_id}`. Enough to
debug a single-request system and to prove the parallel fan-out claim
above, without standing up infrastructure the workload doesn't need.

**Three delivery surfaces, one analysis engine.** The same agent
pipeline is consumable as a REST API, a static web page, an MCP tool
callable from Claude Desktop or any other MCP-compatible client, and
an optional Telegram message. None of these surfaces duplicate agent
logic — they're thin wrappers around `/analyse`, and a failure in any
one of them (e.g. a Telegram send failing) never affects the others.
See [`specs/dev.md`](specs/dev.md#delivery-surfaces).

---

## Quick Start

```bash
git clone https://github.com/<you>/marketlens.git
cd marketlens
cp .env.example .env   # add your OPENROUTER_API_KEY
docker compose up
```

```bash
curl -X POST http://localhost:8080/analyse \
  -H "Content-Type: application/json" \
  -d '{"symbol": "RELIANCE", "period": "3mo"}'
```

Open `http://localhost:8080/docs` for the interactive Swagger UI, or
`http://localhost:8080/` for the static demo page.

To inspect exactly what happened during a run:

```bash
curl http://localhost:8080/trace/<run_id>
```

### Using it from Claude Desktop (MCP)

```bash
cd mcp_server
echo "AGENT_API_BASE_URL=http://localhost:8080" > .env
```

Add to Claude Desktop's local MCP config, restart, then just ask:
"analyse RELIANCE for me." Full setup steps in
[`specs/deploy.md`](specs/deploy.md#connecting-the-mcp-server-sprint-8-local-only).

### Getting a report on Telegram

```bash
curl -X POST http://localhost:8080/analyse \
  -H "Content-Type: application/json" \
  -d '{"symbol": "RELIANCE", "period": "3mo", "notify": "telegram", "chat_id": "<your_chat_id>"}'
```

### Running on Kubernetes

```bash
minikube start --driver=docker
minikube addons enable ingress
kubectl create secret generic agent-secrets --from-env-file=.env
kubectl apply -f k8s/
minikube tunnel
```

Full deployment steps, CI/CD pipeline, and rollback procedure in
[`specs/deploy.md`](specs/deploy.md).

---

## Tech Stack

| Layer | Choice |
|---|---|
| Agent orchestration | LangGraph (state machine, `Send()` fan-out) |
| Multi-agent critique | AutoGen (conversational recommender ↔ critic) |
| LLM | DeepSeek V4 Flash via OpenRouter (OpenAI-compatible) |
| API | FastAPI |
| Language | Python 3.11, `uv`, `ruff` |
| Testing | pytest, pytest-cov (80% gate), LLM-as-judge evals |
| Container | Docker |
| Orchestration | Kubernetes (HPA, health probes, rolling updates) |
| CI/CD | GitHub Actions |
| Data | yfinance, nsetools, Economic Times RSS, newsdata.io — all free, no scraping |
| Delivery | MCP server (`analyse_stock` tool for AI clients) + Telegram (optional, via MCP) |

---

## Project Structure

```
marketlens/
├── CLAUDE.md          ← spec-driven dev instructions for Claude Code
├── specs/
│   ├── product.md     ← what's being built, acceptance criteria, sprint plan
│   ├── dev.md          ← architecture, agent design, every "why" decision
│   ├── test.md         ← test strategy, eval suite, coverage, ruff
│   └── deploy.md       ← Docker, Kubernetes, CI/CD, cost
├── agents/             ← one file per agent, orchestration only
├── tools/              ← pure functions, no LLM calls, fully unit tested
├── graph.py            ← LangGraph graph definition
├── observability/      ← structured logging + request tracing
├── api/                ← FastAPI routes
├── static/             ← single-page demo UI, no framework
├── delivery/           ← Telegram report delivery, decoupled from agent logic
├── mcp_server/         ← exposes MarketLens as an MCP tool for AI clients
├── tests/               ← unit / integration / evals / fixtures
└── k8s/                 ← Kubernetes manifests
```

This project was built using Spec-Driven Development — every
architectural decision was written down and reasoned about in
`specs/` before any code was written. See [`CLAUDE.md`](CLAUDE.md)
for the full workflow.

---

## What I'd Add Next

A few things were deliberately left out of scope, with reasoning
documented in `specs/dev.md` — noted here as a forward-looking roadmap
rather than a gap:

- **Self-hosted inference (vLLM)** — not needed for this project's
  workload (public data, low volume, no fine-tuning), but the same
  OpenAI-compatible client abstraction in `config.py` means swapping
  in a self-hosted backend is a one-line config change, not a rewrite.
- **Prometheus / Grafana** — appropriate once this handles sustained
  production traffic and "is the system healthy over time" becomes
  the relevant question. The per-request tracer answers today's
  actual question — "what happened in this one run" — without
  standing up infrastructure the current workload doesn't justify.
- **A real frontend** — the static demo page proves the API works
  end to end; a proper UI would be the next investment once the
  agent layer is stable.

---

## Disclaimer

**MarketLens produces research reports, not trading advice. Nothing it
outputs should be treated as a recommendation to buy or sell any
security. Always do your own research.**

---
