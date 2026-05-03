# OptimEngine

**Production-grade mathematical optimization service for industrial operations and autonomous AI agents.**

11 solvers covering scheduling, routing, packing, risk-aware planning, and prescriptive intelligence. Built on Google OR-Tools CP-SAT v9.0.0. Live in production with REST API + dual-stack MCP interfaces. Designed as decision-grade infrastructure for AI-native operations.

🌐 **Live API**: [optim-engine-production.up.railway.app](https://optim-engine-production.up.railway.app)
📊 **Live observability**: [public Grafana dashboard](https://optimengine.grafana.net/public-dashboards/21137ba340fc4b6e917a4b108db3e109) — solver invocations, status mix, p95 latency (24h rolling)
✍️ **Technical writing**: [michelecampi.github.io](https://michelecampi.github.io)
👤 **Author**: [Michele Campi](https://github.com/MicheleCampi) — Operations Intelligence Engineer

> This repository is documentation and architecture overview of OptimEngine. The implementation source is private. The service is publicly callable via API key.

---

## What it does

OptimEngine answers four classes of operations questions that mid-market companies typically can't answer well:

**1. What's the optimal schedule for next week?**
Flexible job-shop scheduling (FJSP) with sequence-dependent setup times, per-machine durations, availability windows, quality gates, and yield rates. Returns provably optimal schedules with full audit trail.

**2. What's the optimal route for tomorrow's deliveries?**
Vehicle routing with time windows (CVRPTW), capacity constraints, and per-vehicle eligibility. Returns optimal multi-vehicle assignments minimizing distance, time, or weighted objectives.

**3. How robust is this plan to uncertainty?**
Monte Carlo simulation with CVaR risk metrics, parametric sensitivity analysis with elasticity scores, robust optimization for worst-case and percentile scenarios. Quantifies risk premium beyond what point-estimate planning can express.

**4. What should we do, given history and forecast?**
Prescriptive intelligence: time-series forecasting (exponential smoothing, linear trend, seasonal decomposition) feeds directly into optimization, producing recommendations across conservative/moderate/aggressive scenarios with quantified trade-offs.

---

## Architecture

OptimEngine is built as a 4-layer system where each layer has a single responsibility.

```
┌───────────────────────────────────────────────────────────┐
│  Layer 4 — Discovery                                      │
│  MCP registries (Smithery) · x402 directories · ACP      │
└───────────────────────────────────────────────────────────┘
                            ▲
┌───────────────────────────────────────────────────────────┐
│  Layer 3 — Payment Gateways (thin proxies)                │
│  x402 on Base · Circle Nanopayments on Arc                │
│  Solana gateway · ACP seller runtime                      │
└───────────────────────────────────────────────────────────┘
                            ▲
┌───────────────────────────────────────────────────────────┐
│  Layer 2 — Core Gateway (business orchestration)          │
│  10 modular endpoints · payload validation                │
│  Single source of truth for business logic                │
└───────────────────────────────────────────────────────────┘
                            ▲
┌───────────────────────────────────────────────────────────┐
│  Layer 1 — Raw Solvers (this repository documents)        │
│  FastAPI app · OR-Tools CP-SAT · FastMCP                  │
│  REST API + MCP dual-stack (SSE + Streamable HTTP)        │
└───────────────────────────────────────────────────────────┘
```

**Layer separation matters**: payment formats (x402, Circle, ACP) are isolated from business logic; business logic is isolated from raw solver math. Adding a new chain or marketplace means writing a thin gateway, not duplicating logic.

---

## The 11 solvers, organized by intelligence level

### L1 — Deterministic optimization

| Solver | What it does |
|---|---|
| **`optimize_schedule`** | Flexible job-shop scheduling (FJSP) — provably optimal makespan with setup times, quality gates, machine availability |
| **`optimize_routing`** | Vehicle routing with time windows (CVRPTW) — multi-vehicle, capacity constraints |
| **`optimize_packing`** | Bin packing — single and multi-dimensional |

### L2 — Uncertainty quantification

| Solver | What it does |
|---|---|
| **`analyze_sensitivity`** | Parametric sensitivity with elasticity scores per input |
| **`optimize_robust`** | Robust optimization across worst-case / percentile scenarios |
| **`optimize_stochastic`** | Monte Carlo simulation with CVaR risk metrics — quantifies tail risk |

### L2.5 — Multi-objective trade-offs

| Solver | What it does |
|---|---|
| **`compute_pareto`** | Pareto frontier generation across competing objectives with trade-off analysis |

### L3 — Prescriptive intelligence

| Solver | What it does |
|---|---|
| **`prescriptive_advise`** | Forecast → optimize → risk assessment → ranked recommendations |
| **`forecast_basic`** | Time-series forecasting (exponential smoothing, linear trend, seasonal) — composed at Core Gateway |
| **`risk_analysis`** | Multi-scenario risk evaluation with conservative/moderate/aggressive profiles — composed at Core Gateway |

> **Tool inventory note**: The Layer 1 raw solver service exposes 9 tools directly (visible at the [live API root](https://optim-engine-production.up.railway.app)). 2 additional workflow-level tools (`forecast_basic`, `risk_analysis`) are composed at the Layer 2 Core Gateway for production use cases that combine multiple solvers. Total: 11 capabilities accessible via the full OptimEngine stack.

---

## Stack

- **Solver core**: Google OR-Tools CP-SAT v9.0.0
- **Runtime**: Python 3.12, FastAPI, FastMCP
- **Auth**: ScaleKit OAuth 2.1 (for MCP `/mcp/v2` Streamable HTTP)
- **Validation**: Pydantic
- **Hosting**: Railway
- **Monitoring**: rate-limited endpoints, API key gating

---

## Performance

Tested on realistic mid-market scenarios:

- **Scheduling**: 8 customer orders × 6 production lines, sequence-dependent setup times → provably optimal solution in **10 milliseconds**
- **Stochastic CVaR analysis**: 100 Monte Carlo scenarios on the same problem → **CVaR 95% computed in ~2 seconds**
- **Sensitivity analysis**: 12 parameters × 5 perturbation levels → elasticity scores in **<500ms**

These are not marketing numbers. They're solver runtimes on instances with the structure mid-market manufacturers actually deal with weekly. Detailed case studies are published on [the blog](https://michelecampi.github.io).

Real-time production metrics are available via the [public Grafana dashboard](https://optimengine.grafana.net/public-dashboards/21137ba340fc4b6e917a4b108db3e109) — read-only, no login required, refreshes every 60 seconds.

---

## Interfaces

OptimEngine exposes the same solvers through two protocols.

### REST API

Standard JSON over HTTP. Best for traditional integrations, server-to-server workflows, and non-agent consumers.

```
POST /v1/optimize/schedule
POST /v1/optimize/routing
POST /v1/optimize/stochastic
POST /v1/analyze/sensitivity
POST /v1/compute/pareto
POST /v1/forecast/demand
POST /v1/predict/strategy
[...]
```

API key authentication. Rate-limited.

### MCP (Model Context Protocol)

Tool manifests automatically discoverable by MCP-compatible AI agents (Claude Desktop, Cursor, autonomous agents).

```
/mcp     → SSE transport, open access, rate-limited
/mcp/v2  → Streamable HTTP, OAuth 2.1 protected
```

Registered on Smithery with 9 tools and full server-card.json. Agents can discover OptimEngine, read tool schemas, and invoke solvers in natural language without prior integration.

---

## Why MCP

Most optimization services are REST APIs. OptimEngine is REST + MCP because the audience is shifting.

REST APIs are universal but **require human integration** — a developer wires the endpoint into the agent's context manually.

MCP flips this: an MCP server publishes a tool manifest, and any MCP-compatible agent can discover, read, and call it autonomously. No human integration step.

For a solver service whose target user is "any AI agent that might ever need optimization," the MCP path is where the audience is going. REST stays for traditional integrations.

---

## Use cases — what people actually call OptimEngine for

**Manufacturing operations**
Production schedulers asking "what's the optimal order sequence given setup times and quality constraints?" — typical use across packaging, food processing, cosmetics, light electronics. Mid-market plants (€5-50M revenue) that don't run enterprise APS systems but need decision-grade scheduling.

**Logistics and routing**
Last-mile delivery companies asking "what's the optimal vehicle assignment given today's customer time windows?" — CVRPTW solved deterministically in seconds, not eyeballed by a dispatcher.

**Risk-aware planning**
CFOs asking "what's the right buffer to commit to a customer delivery date, given historical variability?" — Monte Carlo CVaR quantifies the answer in dollars, not in gut feeling.

**AI agents needing decision support**
Autonomous agents asking "given these constraints and uncertainties, what's the right action?" — discoverable via MCP, callable in natural language, returns structured optimization output the agent can act on.

---

## Calling OptimEngine

The service is publicly callable via API key. To request access:

📧 **michele.campi@outlook.com** — describe the use case briefly; access provisioning is manual

Pricing for autonomous agents is per-call via x402 (USDC on Base, Solana, Arc). Pricing for human consumers is via consulting engagements that include OptimEngine usage.

---

## Related work

- **Technical articles**: [michelecampi.github.io](https://michelecampi.github.io) — case studies with real solver output, including a week of contract packaging schedules and AI assistant vs constraint solver comparisons
- **x402 gateway on Arc**: [optim-arc-v3.vercel.app](https://optim-arc-v3.vercel.app) — Circle Nanopayments integration for autonomous agent payments
- **Author profile**: [Michele Campi](https://github.com/MicheleCampi) — Operations Intelligence Engineer, 9+ years in industrial controlling extended into modern computational infrastructure

---

## License

This documentation repository is MIT licensed. The OptimEngine service is proprietary; usage is governed by the API key terms.

---

*OptimEngine is built and operated by [Michele Campi](https://michelecampi.github.io). For consulting engagements that use OptimEngine as part of broader operational analysis, see the [services page](https://michelecampi.github.io/services/).*
