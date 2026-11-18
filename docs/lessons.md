# 6 · Lessons learned

Nine months of production operation, distilled. Some of these I learned from design review feedback; others from pager-worthy incidents that were (thankfully) mine to debug.

## 1. Agent latency is an architecture input, not an SLO footnote

Complex investigations legitimately take minutes — the agent makes many sequential tool calls (correlated alerts, deployments, service health, metrics, logs, host-level diagnostics). That single fact cascades: the model timeout must be tuned for p95 investigations, the sidecar timeout must exceed the model timeout, the retry lease must derive from that same number, and worker concurrency bounds must absorb long-tail latencies. Every number in the chain must come from the same p95, or one link becomes a 3 AM surprise.

## 2. Prompt-only changes are still production changes — with prompt-review discipline

Instructing the agent to *execute* read-only steps (rather than recommend them) transformed investigation behavior with a zero-code diff. Great agility, but it means Markdown files need release discipline: PR review, staged promotion, and post-change verification — exactly like config. The inverse lesson came from the SSH case: when the model listed host diagnostics as "recommended actions" instead of running them, the durable fix was one clarifying paragraph — behavior bugs and code bugs deserve the same root-cause treatment.

## 3. Memory is untrusted input, even your own

The memory store contains what the *model* concluded from conversations — so it can be subtly wrong, stale, or shaped by prompt injection from the threads it summarizes. Rules that emerged:

- Learnings are tagged and auditable (source thread, alert, service), never silently trusted.
- The evaluation harness never uses model-generated memory as ground truth — that's how one bad extraction becomes a self-fulfilling benchmark.
- Extraction runs in an isolated session with a structured-JSON contract and no persistence authority: the sidecar, not the model, decides what gets stored.

## 4. Autonomy is per-action-class, not per-agent

"Is the agent allowed to do X?" was the wrong question. The working model is a matrix: *act* (read-only), *gate* (mutations — buttons, re-verification, TTL), *propose* (draft PRs, vetted allowlist, draft-only). The gate mechanics matter more than the policy: first-click-wins atomic claims, re-verification against live state before every execution, and fail-closed endpoints when auth is unconfigured. Capability ≠ authority remains the most useful sentence in the whole codebase.

## 5. Eval pipelines are the hard part — build them like you build prod

Group-aware splits, an exposure ledger with locking, labels that require independent human review, digests over changing candidates, private staging with file-level paranoia. It's more code than the agent's own plumbing — and it's what converts "I think the new prompt is better" into a measurable, reviewable claim. The corollary: a completed evaluation is *never* authorization to change production; it's one input into a human-reviewed change.

## 6. Concurrency changes semantics — budget it like a feature

Making workers parallel surfaced three separate correctness requirements: semaphore bounds (an unbounded pending-poller parallelism was a self-DoS waiting to happen), WAL mode (concurrent readers during writes), and atomic claims (no double-processing). Sequential code that "will obviously be fine in parallel" almost never is. The safe pattern: bound, transact, then parallelize.

## 7. Startup races are an agent-system genre

Every multi-process agent system has the same first-day bug: background loops outrun the heavyweight gateway that boots slowly. The fix generalizes — explicit readiness gates, early-exit on startup crash, transient-connection retry so a single blip during deploys doesn't become a false incident. Apply it between *any* two processes with asymmetric boot times.

## 8. Make failure legible

The system's failure messages distinguish `connection refused` (deploy blip) from `upstream timeout` (model stall) because they page different people. Handoffs report explicitly when nothing was confirmed instead of assuming delivery. Failed Slack posts return 502 so the upstream platform retries instead of silently dropping. An agent that lies about its own failure state is worse than no agent — designing error paths as product surfaces, not afterthoughts, is what made the team trust it enough to use it.

## 9. The changelog is your roadmap's memory

Keeping an honest Keep-a-Changelog file (even for an internal system) is how I can now explain *why* the retry queue has a `processing` column, *why* the sidecar is a package, and *why* the pending poller checks instance-level labels. The timeline table in this repo's README is literally the changelog condensed — that's not vanity; it's institutional memory that outlives incidents.

## 10. Don't confuse a working demo with a working system

The demo-to-production distance is measured in: idempotency, bounded caches, credential blast-radius analysis, lease semantics, TTL GC, health counters, and eval discipline. None of those are agent magic — all of them are engineering. The LLM was maybe 10% of the code; the other 90% is what makes it production-safe. That ratio is the actual job.
