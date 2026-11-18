# 3 · Learning loop

An agent that treats every alert like the first one it has ever seen will stay junior forever. The learning loop is what compounds: the agent investigates before pages fire, records what humans correct, and walks into the next incident with those corrections already loaded.

## A · Pre-fire investigation (act before the page)

Alert rules have a `for` duration: the condition must hold that long before the alert *fires* and pages anyone. That pending window is free information, so the sidecar polls the rules API (not the alert manager — pending alerts live in rule evaluation state, not in the alerting layer) and starts an investigation when a pending alert has been true for **≥70% of its `for` duration**.

Key mechanics:

- **SQLite correlation**: each pending alert is stored under a key built from stable Prometheus labels (`alertname`, `service`, `instance`, `cluster`, `namespace`, …). When the same alert later fires and arrives via the alert platform, the sidecar matches on labels and **reuses the existing Slack thread** instead of creating a second one — the human sees one continuous story: prediction → (if it fired) FIRED → investigation.
- **Dedup + TTL**: each pending alert is investigated once; records older than 2 hours are garbage-collected.
- **Framing discipline**: a pending alert is a *prediction, not proof*. The agent investigates with full depth — including host-level diagnostics — but frames findings as "if this fires, the likely cause is X". Many pending alerts simply resolve on their own; detecting self-healing is part of the value.
- **Atomic fire transition**: `UPDATE ... WHERE fired = 0 RETURNING` guarantees a pending investigation can't be double-promoted by a race between pollers.

What this buys: by the time a page arrives, there is usually an investigation thread already underway — sometimes the root cause is already identified. The pager still goes to a human; the *latency to diagnosis* collapses.

## B · Closure learning (corrections become memory)

Background monitor classifies alert threads as done and extracts learnings:

| Trigger | Meaning |
|---|---|
| Alert resolved + 5 min grace | The primary path — wait briefly for final human notes, then extract |
| Thread quiet for 60 min | Fallback when no resolved event arrives |

Then, based on the thread's origin and participation:

| Thread origin | Human messages? | Extracted |
|---|---|---|
| Firing | Yes | **Human correction** — what did the human say differently? |
| Pending | Yes | Human correction |
| Pending | No | **Prediction accuracy** — was the pre-fire analysis right? |
| Firing | No | Skip (nothing to learn) |

Extraction runs in an **isolated session** (a dedicated session key, so the extraction model can't accidentally reuse or pollute the live alert's context), returns structured JSON only, and is deliberately stripped of persistence authority: the extraction can't write files, claim memory retention, or retain by itself. The sidecar retains the result to the durable memory store with structured tags (`alert_name`, `service`, `human_correction`/`prediction_accuracy`) and posts a small `:brain:` confirmation in the thread **only after** the memory API synchronously confirms one stored item. Failed extractions retry on the next cycle; the Slack transcript is never modified as a side effect.

## C · Recall before investigate

Before investigating an alert from scratch, the agent queries memory for prior corrections on the same alert/service. A recent correction reshapes the investigation: start from the corrected root cause, verify it still applies, and only then expand. This is the difference between an agent that has been on the job for months and one that resets every morning.

## D · Self-improvement

Two tiers:

1. **Curated memory.** After each investigation, the agent updates its curated knowledge file (patterns, false positives, quirks, maintainer info). Human-reviewed, versioned, small.
2. **Configuration proposals.** When a learning is structural (recurring pattern, wrong runbook, a new rule), the agent opens a draft PR against its own repository — same vetting stack as any other proposal ([doc 2](safety.md)). Fully autonomous self-improvement (agent merges its own prompt changes) is intentionally **not** enabled; the feedback loop closes through human review.

## Why not just fine-tune?

The learnings here are *operational context*, not model capability: which alert is a false positive, which service silently redistributes load, which datasource has a known gap. Retrieval over a curated, tagged memory store beats fine-tuning for this — it's auditable (every memory has a source thread), correctable (a bad correction can be removed), and immediately effective (no training cycle). The evaluation harness's job ([doc 4](evaluation.md)) is to make sure these learnings actually improve decisions rather than encode one bad day.
