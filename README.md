---
title: CloudFinOps Env
emoji: ☁️
colorFrom: blue
colorTo: green
sdk: docker
pinned: false
app_port: 8000
base_path: /web
tags:
  - openenv
---
 
<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-3776AB?style=flat-square&logo=python&logoColor=white" />
  <img src="https://img.shields.io/badge/FastAPI-0.111-009688?style=flat-square&logo=fastapi&logoColor=white" />
  <img src="https://img.shields.io/badge/OpenEnv-compatible-22c55e?style=flat-square" />
  <img src="https://img.shields.io/badge/Tests-84_passed-22c55e?style=flat-square" />
  <a href="https://github.com/Fnc-Jit/Cloud-Solutions_Re/actions/workflows/validate.yml">
    <img src="https://github.com/Fnc-Jit/Cloud-Solutions_Re/actions/workflows/validate.yml/badge.svg?style=flat-square" />
  </a>
  <img src="https://img.shields.io/badge/License-MIT-6366f1?style=flat-square" />
</p>
<h1 align="center">CloudFinOps-Env</h1>
 
<p align="center">
  RL environment for cloud cost optimization, SLA incident management, and carbon emissions reduction — with 4 difficulty tiers, live dashboard, and explainable per-step scoring.
</p>
<p align="center">
  <b>Built for the Meta AI × Hugging Face OpenEnv Hackathon</b>
</p>
---
 
![CloudFinOps Dashboard](assets/dashboard.png)
 
---
 
## TLDR
 
> Real cloud teams constantly trade off cost, SLA risk, and sustainability under time pressure. No existing RL benchmark captures that three-way tradeoff with realistic constraints. CloudFinOps-Env builds that benchmark: an agent manages a fleet of AWS-style servers across 4 task tiers — from zombie cleanup to carbon migration — and is scored on cost efficiency, uptime, carbon reduction, and stakeholder communication simultaneously.
>
> **Baseline agent (Llama 4 Scout 17B): 0.7861 average score across all 4 tiers. 84 unit tests. Full CI/CD. Live dashboard at `/dashboard`.**
 
---
 
## Motivation — why this environment exists
 
Cloud infrastructure decisions are not made in isolation. Every action an SRE or FinOps engineer takes involves at least three competing objectives at once:
 
- **Cost** — unused capacity is money burning. Over-provisioning is the default failure mode.
- **Reliability** — a SLA breach is an incident. Aggressive cost-cutting that kills uptime is worse than doing nothing.
- **Sustainability** — carbon emissions from cloud infrastructure are increasingly a real business metric, not just PR.
Most existing RL environments for infrastructure management optimize one objective and treat the others as constraints or ignore them entirely. That maps poorly to real operations, where the whole point is navigating the tradeoff.
 
CloudFinOps-Env is designed to **benchmark whether an agent can make the same multi-objective tradeoff that human operators make every day** — with the same delayed consequences, the same stakeholder communication overhead, and the same budget pressure. It was built for the OpenEnv hackathon and for anyone who wants a realistic stress test for infrastructure decision agents.
 
---
 
## Architecture
 
```
┌────────────────────────────────────────────────────────────────┐
│                       CloudFinOps-Env                          │
│                                                                │
│  ┌──────────┐    ┌───────────────────┐    ┌─────────────────┐ │
│  │ models.py│───▶│  engine +         │◀───│   server/app.py │ │
│  │ Pydantic │    │  environment.py   │    │ FastAPI+OpenEnv  │ │
│  │ Schemas  │    │                   │    │ + Dashboard      │ │
│  └──────────┘    └───────────────────┘    └─────────────────┘ │
│                          ▲                        ▲            │
│                          │                        │            │
│                    ┌─────┴─────┐          ┌───────┴──────┐    │
│                    │   Tests   │          │ inference.py  │    │
│                    │ 84 unit   │          │  LLM Agent    │    │
│                    │   tests   │          │  Evaluator    │    │
│                    └───────────┘          └──────────────┘    │
└────────────────────────────────────────────────────────────────┘
```
 
---
 
## Environment Description
 
CloudFinOps-Env simulates day-to-day cloud operations where an agent manages a fleet of AWS-style servers. Each episode gives the agent a fixed budget and a set of objectives. The agent must observe server state, decide which actions to take at each step, and avoid SLA breaches and budget overruns — while reducing cost, carbon, or both depending on the task tier.
 
### What the agent sees
 
A structured observation delivered via `POST /reset` and refreshed after each `POST /step`:
 
```json
{
  "servers": [...],
  "traffic_load": 30.0,
  "spike_detected": false,
  "incidents": [],
  "budget_remaining": 5.0,
  "time_step": 0,
  "inbox": ["Ops Team: Please confirm you've reviewed the scaling plan."],
  "carbon_kwh": 0.0
}
```
 
Each server includes `cpu_history` and `memory_history` — the last 3 steps — so agents can detect trends without external memory systems.
 
### What the agent does (action space)
 
| Command | Effect | Reward |
|---|---|---|
| `TERMINATE` | Kill a server immediately | +10 |
| `UPSCALE` | Queue upgrade for next step | −5 |
| `DOWNSCALE` | Halve cost; CPU load ×1.8 | +5 |
| `REDISTRIBUTE_LOAD` | Spread CPU evenly across fleet | +3 |
| `IGNORE` | No action this step | 0 |
 
**Penalties:**
- Invalid target: −2
- SLA breach (CPU ≥ 100%): −100 + episode ends
- Budget overrun: −20 + episode ends
- Sustained high cost (>$0.50/step): −1/step
### What the agent is scored on
 
Scoring is task-dependent and fully transparent via `reward_breakdown` and `score_breakdown` in the step `info` payload (see [Explainable Scoring](#explainable-scoring)).
 
---
 
## Task Tiers
 
### Easy — "Zombie Cleanup"
Terminate 3 idle servers without touching active ones.
**Budget:** $5.00 | **Servers:** 10 | **Grading:** +1/3 per zombie killed, −0.25 per wrongful termination.
 
### Medium — "CTO Budget Squeeze"
Cut cloud costs by ≥50% across 12 over-provisioned servers.
**Budget:** $10.00 | **Servers:** 12 | **Grading:** Proportional to `cost_saved_pct / 50%`.
 
### Hard — "Black Friday Chaos"
Handle a traffic spike with exponential ramp. Keep DB servers alive under budget pressure.
**Budget:** $4.00 | **Servers:** 8 | **Grading:** Uptime (60%) + Cost Efficiency (40%) + Inbox Bonus.
 
### Green — "The Green Initiative"
Reduce carbon emissions by 40% by migrating workloads from dirty x86 instances to ARM Graviton.
**Budget:** $8.00 | **Servers:** 10 | **Grading:** Carbon Reduction (50%) + Uptime (30%) + Cost (10%) + Inbox (10%).
 
---
 
## Key Design Decisions
 
These choices were deliberate. Each one reflects a specific failure mode in real operations or in standard RL benchmarks.
 
**Deterministic noise via hash-seeded jitter** — Episodes are reproducible. The same seed always produces the same metric trajectory, making benchmark comparisons meaningful rather than variance-dominated.
 
**Delayed scaling (UPSCALE queues for next step)** — Agents cannot react instantaneously. An upscale decision made now affects the next step, forcing planning under uncertainty rather than pure reactive policy.
 
**Trailing metrics history (last 3 steps)** — Designed for LLM agents with limited context windows. Rather than requiring an external memory system, trend detection is baked into the observation. A CPU rising across three consecutive steps is directly visible without any state tracking.
 
**Carbon intensity modeled by architecture** — ARM Graviton (r6g) instances produce 2–3× less carbon per step than equivalent x86 (c5/m5) instances. This models the real-world energy efficiency gap and makes the Green tier a genuine migration planning problem, not just a "terminate stuff" task.
 
**Human-in-the-loop inbox** — Stakeholders send messages; replying earns bonus points. This tests whether an agent can handle communication overhead while managing infrastructure — a real operational requirement that pure cost-optimization benchmarks omit.
 
**Strict score clamping to (0, 1)** — All scores are epsilon-clamped (0.001 ≤ score ≤ 0.999). No perfect scores, no zero scores. Forces agents to demonstrate partial competence gradients rather than binary pass/fail.
 
---
 
## Baseline Results
 
```
============================================================
  BASELINE — Llama 4 Scout 17B, all 4 tiers
============================================================
  easy:    0.9990  ██████████████████░░
  medium:  0.4954  █████████░░░░░░░░░░░
  hard:    0.7000  █████████████░░░░░░░
  green:   0.9499  ██████████████████░░
  AVERAGE: 0.7861
============================================================
```
 
![Baseline Evaluation Results](assets/baseline_scores.png)
 
The medium tier score (0.4954) reflects the genuine difficulty of the CTO Budget Squeeze: cutting 50% costs on 12 servers without triggering SLA breaches requires planning more than 1 step ahead, and the baseline agent's greedy policy degrades under that constraint. The hard tier (0.7000) recovers because the exponential traffic ramp is detectable in the trailing metrics history — agents that read `cpu_history` trends before acting outperform reactive ones.
 
> All scores are within the open interval (0, 1). Run your own evaluation by following the setup instructions below and pointing the evaluator at any OpenAI-compatible endpoint.
 
---
 
## Carbon Intensity Reference
 
| Instance | Architecture | Carbon (kWh/step) | Class |
|---|---|---|---|
| `t3.micro` | x86 | 0.005 | Web |
| `t3.medium` | x86 | 0.012 | Web |
| `t3.large` | x86 | 0.022 | Web |
| `c5.large` | x86 | **0.035** | Compute |
| `c5.xlarge` | x86 | **0.065** | Compute |
| `r6g.medium` | ARM Graviton | 0.008 | DB |
| `r6g.large` | ARM Graviton | 0.015 | DB |
| `r6g.xlarge` | ARM Graviton | 0.028 | DB |
| `m5.large` | x86 | **0.040** | Batch |
| `m5.xlarge` | x86 | **0.075** | Batch |
 
ARM Graviton instances produce **2–3× less carbon** than equivalent x86. The Green tier is won by migrating compute and batch workloads from c5/m5 to r6g before time runs out.
 
---
 
## Explainable Scoring
 
Every step returns a `reward_breakdown` in `info`. The final step (when `done=true`) returns a full `score_breakdown`. This makes the grading contract auditable — evaluators can see exactly which component drove the score, not just the final number.
 
```json
{
  "grader_score": 0.7421,
  "reward_breakdown": {
    "penalties": { "high_runtime_cost": 3.0, "upscale_action_cost": 10.0 },
    "bonuses": { "terminate_action": 20.0, "inbox_reply": 2.0 }
  },
  "score_breakdown": {
    "task_id": "hard",
    "raw_score": 0.7421,
    "final_score": 0.7421,
    "cost": { "saved_pct": 0.3814, "weight": 0.4, "contribution": 0.1526 },
    "sla": { "breached": false, "weight": 0.6, "contribution": 0.6 },
    "carbon": { "reduction_pct": 0.0, "weight": 0.0, "contribution": 0.0 },
    "inbox_reply_bonus": { "applied": false, "value": 0.0 },
    "penalties": { "grading": { "active_termination": 0.0, "sla_crash_penalty": 0.0 } }
  }
}
```
 
Implementation: `reward_breakdown` is assembled per step in the engine. `score_breakdown` is attached at episode end via `_build_score_breakdown(...)` in `server/cloudfinops_env_environment.py`.
 
---
 
## Setup
 
**Requirements:** Python 3.10+, Docker
 
Set these three environment variables before running:
 
| Variable | Required | Description |
|---|---|---|
| `API_BASE_URL` | Yes | OpenAI-compatible LLM endpoint |
| `MODEL_NAME` | Yes | Model identifier |
| `HF_TOKEN` | Yes | Hugging Face or provider API key |
 
### Docker (recommended)
 
```bash
git clone https://github.com/Fnc-Jit/Cloud-Solutions_Re.git
cd cloudfinops_env
 
docker build -t cloudfinops-env:latest -f server/Dockerfile .
docker run -p 8000:8000 cloudfinops-env:latest
 
# Open dashboard
open http://localhost:8000/dashboard
 
# Run evaluator against live server
docker run \
  -e ENV_BASE_URL=http://host.docker.internal:8000 \
  cloudfinops-env:latest python3 /app/env/inference.py
```
 
### Local (uv)
 
```bash
uv sync
uv run server
# or: uvicorn server.app:app --reload --host 0.0.0.0 --port 8000
```
 
### Local (pip)
 
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install "openenv-core[core]>=0.2.2" pytest
pip install -e .
python inference.py
```
 
### Python SDK
 
```python
from cloudfinops_env import CloudFinOpsAction, CloudFinOpsEnv
 
with CloudFinOpsEnv(base_url="http://localhost:8000") as env:
    result = env.reset()
    print(f"Budget: {result.observation.budget_remaining}")
 
    result = env.step(CloudFinOpsAction(command="TERMINATE", target_id="idle-0"))
    print(f"Reward: {result.reward}")
```
 
---
 
## API Reference
 
| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/reset` | Reset for a task (`{"task_id": "easy"}`) |
| `POST` | `/step` | Submit action, advance engine |
| `GET` | `/state` | Current observation (read-only) |
| `GET` | `/schema` | Action/Observation JSON schemas |
| `WS` | `/ws` | WebSocket persistent session |
| `GET` | `/dashboard` | Real-time live dashboard |
| `GET` | `/history` | Action history for current episode |
 
---
 
## Testing
 
84 unit tests across 20 test classes. Covers engine mechanics, scoring, edge cases, and deterministic replay.
 
```bash
# Docker
docker run --rm cloudfinops-env:latest python3 -m pytest tests/ -v --tb=short
 
# Local
python3 -m pytest tests/ -v --tb=short
```
 
| Test Class | Tests | Coverage area |
|---|---|---|
| `TestReset` | 6 | Clean state, all 4 tasks, invalid task handling |
| `TestDeterministicNoise` | 3 | Reproducibility, seed isolation, amplitude bounds |
| `TestActions` | 9 | All 5 commands + inbox reply |
| `TestSLABreach` | 1 | Breach detection, episode termination |
| `TestGrading` | 4 | All 4 graders, score ranges, carbon scoring |
| `TestCarbonTracking` | 4 | Accumulation, post-terminate reduction, catalog coverage |
| `TestTrailingHistory` | 3 | Initial values, growth, max depth enforcement |
| `TestEpisodeBoundaries` | 3 | Max steps, budget overrun, post-done behavior |
| `TestCascadingFailures` | 3 | Redistribution after terminates → SLA breach risk |
| `TestBudgetEdgeCases` | 4 | Zero budget, negative budget, burn rate |
| `TestInvalidActions` | 6 | Double-terminate, ghost servers, null target |
| `TestDownscaleSafety` | 3 | 1.8× CPU multiplier, cost halving, breach risk |
| `TestGreenOpsEdgeCases` | 4 | Carbon rate tracking, ARM vs x86 |
| `TestInboxMechanics` | 6 | Reply bonus, clearing, empty reply |
| `TestOptimalPlaySequences` | 4 | Known-optimal sequences per task type |
| `TestDeterministicReplay` | 3 | Same actions → same scores |
| `TestGraderBoundaries` | 5 | Score clamping, zero-score conditions |
| `TestUpscaleQueueTiming` | 4 | Queue behavior, next-step application |
| `TestTrafficSimulation` | 4 | Logarithmic growth (hard), spike detection |
 
---
 
## CI/CD
 
GitHub Actions runs on every push and PR:
 
1. Unit tests (`pytest tests/ -v`)
2. Syntax check (AST parse all Python files)
3. OpenEnv spec validation (`openenv.yaml` has ≥3 tasks)
4. Docker build + container smoke test
---
 
## Project Structure
 
```
cloudfinops_env/
├── __init__.py                          # Module exports
├── models.py                            # Pydantic schemas (openenv base types)
├── client.py                            # CloudFinOpsEnv SDK client
├── openenv.yaml                         # OpenEnv manifest (spec_version: 1)
├── pyproject.toml
├── README.md
├── inference.py                         # LLM baseline evaluator
├── server/
│   ├── cloudfinops_env_environment.py   # Physics engine + OpenEnv wrapper
│   ├── app.py                           # FastAPI app via openenv create_app()
│   ├── dashboard.html                   # Real-time dashboard
│   ├── Dockerfile
│   └── requirements.txt
├── tests/
│   └── test_engine.py                   # 84 pytest unit tests
└── assets/
    ├── dashboard.png
    └── dashboard_details.png
```
 
---
 
## Tech Stack
 
| Layer | Technology |
|---|---|
| Backend | Python 3.10, FastAPI, OpenEnv core |
| RL environment | Custom physics engine (environment.py) |
| Schemas | Pydantic v2 |
| Frontend | HTML5, CSS3, JavaScript (dashboard) |
| Container | Docker multi-stage build |
| CI | GitHub Actions |
 
---
 
## Roadmap
 
- [ ] Continuous traffic patterns (sine wave, diurnal cycle) beyond current step-function model
- [ ] Multi-agent mode — separate cost, SLA, and carbon agents that must coordinate
- [ ] Real AWS pricing API integration for dynamic cost modeling
- [ ] Extended carbon model with regional grid intensity (us-east-1 vs eu-west-1)
- [ ] Persistent episode history for cross-run benchmarking
---
 
## Related Projects
 
- **[God's Eye](https://github.com/Fnc-Jit/Gods-Eye)** — Autonomous multi-agent SIEM platform
- **[MissionCtrl](https://github.com/Fnc-Jit/MissionCtrl-MultiAgent-Orchastrator)** — AI oversight fleet environment for hallucination detection in multi-agent systems
---
 
## License
 
MIT
