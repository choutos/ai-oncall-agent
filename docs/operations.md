# 5 · Operations

The unglamorous half of agent engineering, where the actual pager-nightmares are prevented. Everything below was learned the hard way, in production.

## Deployment shape

- **Kubernetes** — one pod per environment (dev + prod), managed as a Kustomize overlay promoted by ArgoCD. Agent config, model endpoint, and all credentials arrive from Vault-sourced secrets; the deployment repo owns infrastructure config, the source repo owns agent behavior.
- **State that matters lives on PVC-backed SQLite**: retry queue, action ledger, pending-alert correlation. Pod restarts must not lose alerts.
- **Background loops** run inside the sidecar (asyncio): alert poller (30s), pending poller (60s), closure monitor (120s), retry worker (15s), action TTL GC (300s).

## Reliability engineering, agent edition

**The startup race.** The gateway (Node) takes a while to boot — it loads ~140 skills, the Slack plugin, memory, and the model client. The sidecar's pollers start within seconds and would fire the first alert of the day straight into `ECONNREFUSED`. Fix: the entrypoint waits for the gateway to accept TCP connections (configurable timeout, crash-fast if it dies during startup) *before* launching the sidecar — plus a short transient-connection retry with backoff in the call path, so one blip during a deploy doesn't surface as a scary Slack error.

**The retry queue bug (23 min → 3 min).** Two features shared one column: the retry *lease* and the *backoff schedule* both lived in `next_retry_at`. The lease (460s, designed to prevent double-processing after a pod crash) kept overwriting the exponential backoff (0s/60s/120s), so retries ran at lease-expiry pace: ~23 minutes per failed alert. The fix moved concurrency control to a separate `processing` column and let backoff keep its own timestamps. Same failure mode in reverse: a *removed* lease also needs a re-test of the crash-recovery path, which is why the column rename went through review with both scenarios called out.

**Concurrency without thundering herds.** Sequential workers were a scaling bug in disguise: 5 queued retries × 400s timeout ≈ 33 minutes of dead air. Parallelizing required three guardrails: per-worker semaphores (default 5 concurrent investigations), SQLite in WAL mode (readers during writes, no fsync-per-commit stalls), and atomic claims on queue rows so two workers can't grab the same item. The pending poller gained the same semaphore after shipping without one — parallelism added without a bound is a self-DoS.

**Timeouts that match the workload.** Deep investigations (Alertmanager + Consul + Loki + ArgoCD + logs) take 5+ minutes under model load. The sidecar timeout and the gateway's model timeout were tuned *together* (400s→600s both sides) because the pair, not either alone, governs the user-visible failure — and the retry lease is derived from the same number so tuning one can't orphan the other.

**Bounded everything.** In-memory caches have explicit caps and eviction (thread map: 2048 entries; webhook idempotency cache; pending records: 2h TTL; actions: 24h TTL). Long-running agents leak; only explicit bounds stop it.

**Rate limits as API.** Slack 429s are retried using the server's `Retry-After` header, capped (3 attempts, ≤60s per wait, 120s total); credential and scope errors are never retried. A partial export aborts without replacing the previous good output.

**Honest delivery semantics.** The Slack bridge is at-least-once by construction: an ambiguous failure (timeout after Slack accepted the post) is treated as failure and retried, which can duplicate a parent thread; there is no idempotency key. I chose a rare duplicate over a lost alert, wrote that tradeoff down in the design doc, and set the retry buffer's eviction policy (bounded ring, drop-oldest, logged) explicitly.

## Observability

- `/healthz` exposes per-component counters: queue depth, actions pending/approved/expired, closure-monitor stats (threads eligible/processed, learnings extracted, prediction-accuracy extractions, extraction errors), pending-poller status.
- Structured logs with redaction of secrets; error paths distinguish *"connection refused"* from *"upstream timeout"* — one is a deploy blip, the other is a model-side stall, and they page different people.
- A small usage-metrics collector runs as a supervised third process, aggregating token counts per model/provider from session trajectories (no prompts, no responses) so the cost of an investigation is queryable later.

## The refactor that mattered

The sidecar grew to a 3,300-line `main.py`. v1.12 split it into a package with one module per responsibility (config with all 53 env vars in one place, state, pure utils, Svix verification, gateway client, Slack client, retry queue, API client, pollers, app) — and kept `main.py` as a thin re-export hub so existing imports kept working. Two takeaways:

1. **Shared flow beats repeated flow.** The duplicated "build prompt → call gateway → post to Slack → enqueue retry on failure → post error" sequence became one helper; four call sites, one place to fix races.
2. **Legacy compatibility is cheap if you design for it.** The re-export hub cost ten lines and bought a zero-downtime migration.

## Config surface as engineering surface

Fifty-three environment variables is not "just config" — it's the system's real API. Each has a documented default and a stated reason to exist; each toggle has a tested on/off path. The discipline that keeps it sane: any new capability ships with its env vars, its test, and its healthz counters in the same PR. If a flag can't say what it guards and what happens when it's off, it doesn't ship.
