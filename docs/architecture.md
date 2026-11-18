# 1 · Architecture

## The shape of the system

Three cooperating layers, all running in one Kubernetes pod, one environment per cluster (dev and prod):

```
┌────────────────────────── Agent pod ───────────────────────────┐
│                                                                │
│  ┌──────────────────┐   investigation    ┌──────────────────┐  │
│  │  Agent runtime   │◀───────────────────▶│   A2A sidecar    │  │
│  │  (OpenClaw gw)   │    prompts/verdicts │  (Python/FastAPI)│  │
│  │                  │                     │                  │  │
│  │  ~140 bundled +  │                     │  webhook intake  │  │
│  │  17 custom skills│                     │  dedup + auth    │  │
│  │  memory plugin   │                     │  Slack threads   │  │
│  └──────────────────┘                     │  retry queue     │  │
│         │            │                    │  closure monitor │  │
│         ▼            ▼                    │  pending poller  │  │
│  model gateway   tools (metrics, logs,   └──────────────────┘  │
│  (LiteLLM)       deploys, K8s, SSH)                │           │
│                                                    ▼           │
│                                        SQLite on PVC (queues,  │
│                                        action ledger, pending  │
│                                        correlation)            │
└────────────────────────────────────────────────────────────────┘
```

**Why two processes?** The agent runtime is optimized for reasoning and tool use; it is stateless between sessions except for its memory plugin. The sidecar is optimized for everything that must be *durable and boring*: authentication, deduplication, thread bookkeeping, retries, and scheduled background loops. Splitting them means:

- The sidecar is a normal Python service — unit-testable with mocks, no LLM in the loop. Its test suite alone is ~5,900 lines.
- The agent's prompt configuration can change via PR without touching the plumbing, and vice versa.
- A gateway restart (or an OOM kill) doesn't lose alerts: the sidecar holds the queue, and its retry worker drains once the gateway is healthy again.

## The skills layer

The agent's capabilities are packaged as **skills**: a directory with a `SKILL.md` (procedural instructions, guardrails, gotchas) and small scripts where automation beats prose. The runtime loads ~140 skills in total; 17 are bespoke to this system:

| Skill | What it wraps |
|-------|---------------|
| `alertmanager` | Query active alerts and silences; silences always carry a team filter |
| `argocd` | App sync status, health, deploy history — correlate alerts with recent changes |
| `grafana` | Prometheus/Thanos range + instant queries, Loki log queries and stats, profiling, via one helper script |
| `incidentio` | Incident platform queries over MCP: open incidents, postmortems, follow-ups |
| `sdm` | Zero-trust SSH to hosts, database connections, cluster kubeconfigs |
| `steampipe` | SQL over cloud/K8s/service-discovery state; also the catalog/ownership lookup ("who owns X?") |
| `linear` | Issue inspection; writes require explicit human approval |
| runbook handbook | Search platform + infrastructure handbooks for the alert's runbook |
| internal CLI | Service-discovery health, logs, metrics as a fallback path |
| CI/CD diagnostics | Read-only GitHub Actions runs/jobs/logs for the affected repository |
| DNS tester | Bounded A-record resolution checks for resolver/DNS alerts |
| ownership handoff | Structured handoff of off-team alerts to specialist channels |
| semi-formal reasoning | The diagnosis protocol (below) |
| propose-fix / self-improve | Bounded draft-PR proposals (see [safety](safety.md)) |

**The insight:** an LLM agent's reliability is mostly determined by how well its tool layer encodes the *non-obvious* operational knowledge — which endpoint, which datasource UID, which label conventions, what NOT to touch. Most of that lives in skill markdown next to the scripts, where both the model and human reviewers can read it.

## Config-as-documentation

Agent behavior is defined in Markdown, not code:

- `AGENTS.md` — the investigation workflow: parse → investigate → diagnose → post → propose fix → reflect & improve
- `RULES.md` — hard constraints (never silence without a team filter, never execute a mutation without re-verification, always include evidence)
- `SOUL.md` — communication style: verdict first, then evidence, then actions; severity emoji; no pleasantries
- `TOOLS.md` — per-tool notes and known pitfalls

This is more than docs — it's the interface where engineers (not just the agent author) review and change agent behavior through normal PR review. Two consequences I've learned to respect:

- **Prompt-only changes are still production changes.** Instructing the agent to *execute* read-only steps instead of recommending them changed investigation behavior across the board — no code diff involved.
- **Behavior drift is a regression class.** When the model stopped executing host-level diagnostics via SSH and instead listed them as "recommended actions", the fix was one clarifying paragraph in the agent's instructions — the kind of fix that only works because behavior is centralized in one place.

## The investigation protocol

Diagnoses are not allowed to be vibes. Before posting anything, the agent completes a **reasoning certificate** (from the semi-formal reasoning skill):

1. **Numbered premises** — alert name, rule condition, severity, initial hypothesis (explicitly marked UNVERIFIED).
2. **Mandatory data collection** — all relevant evidence sources queried before interpretation: correlated alerts (±30 min), recent deployments, service health, metrics around the alert window, logs, cluster state.
3. **Execution trace** — each claim tied to a concrete observation with source and time.
4. **Disconfirmation** — at least one competing hypothesis with the discriminating check that would rule it in/out.
5. **Conclusion** — diagnosis with confidence score, blast radius, and recommended actions.

This turns "the agent said so" into "each line is checkable by a reviewer", which is what makes the rest of the safety model (approvals, PR proposals) auditable.

## A2A: agent-to-agent delegation

Some alerts aren't mine to solve. The agent runs a fast **ownership triage** before deep-diving into recurring off-team patterns:

| Signal | Minimum discriminating check | Routed to |
|---|---|---|
| Kubernetes control-plane components failing | Confirm it's a control-plane node/service, not just a workload on the cluster | Compute platform team |
| Storage/network latency on VMs | Distinguish network-path evidence (packet loss, retransmits) from storage health (I/O latency) | Network team vs storage team; joint triage if inconclusive |
| DNS resolution failures | Separate resolver/zone issues from network reachability on the path | Network vs my team, by evidence |

Delegation uses the [A2A protocol](https://a2a-protocol.org): an agent-card for discovery, JSON-RPC for messaging, and mutual authentication through a dedicated gateway for cross-agent traffic. The handoff is *evidence-first*: a short packet with symptoms, scope, read-only observations with timestamps, uncertainty, and the original thread link. The agent is required to say explicitly when a handoff was **not** confirmed — a sent message is not the same thing as a colleague receiving it.

Two rules that came from painful review feedback:

- **Every operation confirms separately.** A successful channel post ≠ accepted task ≠ thread delivered. Report what you verified, not what you assumed.
- **Never page or mutate alert routing** as part of a handoff — that stays with the human process.

## Model layer

The model is served through a LiteLLM-compatible gateway, and the agent has been carried across model upgrades (GLM-5 → 5.1-FP8 → 5.2-FP8) **with zero agent-code changes** — model identity is config, not architecture. Two practical notes from production:

- Complex investigations legitimately take 5+ minutes (multiple tool round-trips). Timeouts must be set for the *p95 investigation*, not the median chat completion — and retry-queue leases must be derived from that same number.
- Smaller/cheaper models change the reasoning-protocol adherence rate, which is exactly why the evaluation harness ([doc 4](evaluation.md)) matters more than model choice.
