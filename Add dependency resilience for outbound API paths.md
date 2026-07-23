Dependency resilience should be designed as a control loop, not just 'add retries'.

What to add for outbound API paths
1. Timeout budget
2. Retry policy
3. Failure classification
4. DLQ and replay handling
5. Observability and guardrails

1. Timeout budget
Goal: prevent threads from waiting indefinitely and consuming worker capacity.

1. Set an end-to-end per-job budget (example: 120s for one outbound step).
2. Split that budget across attempts (example: 3 attempts, each call timeout 20s, plus backoff).
3. Enforce timeout in code with cancellation token per call.
4. Propagate remaining budget to downstream calls so one slow dependency cannot starve the whole workflow.

Recommended pattern
1. callTimeoutPerAttempt: 15-30s for typical control-plane API calls.
2. maxAttempts: 2-3.
3. totalStepBudget: fixed cap (example: 60s).
4. if remaining budget < minimum useful timeout, fail fast as timeout_budget_exhausted.

2. Retry policy
Goal: retry only transient failures, avoid retry storms.

1. Use exponential backoff with jitter.
2. Retry only retriable classes:
- HTTP 408, 429, 5xx
- socket/connect timeouts
- transient DNS/TLS/network errors
3. Do not retry non-retriable classes:
- HTTP 400/401/403/404 (unless business rule says otherwise)
- validation/contract errors
4. Cap retries and add a circuit-breaker style cool-down when a dependency is broadly unhealthy.

Recommended defaults
1. Backoff: 1s, 3s, 7s (with jitter).
2. Max attempts: 3.
3. Stop early if total timeout budget is exceeded.
4. Respect Retry-After header for 429/503 when present.

3. Failure classification
Goal: make decisions deterministic and supportable.

Define a normalized failure taxonomy for every outbound call:
1. transient_timeout
2. transient_throttling
3. transient_network
4. transient_server_error
5. permanent_auth
6. permanent_validation
7. permanent_not_found
8. unknown

For each failed call, record:
1. dependencyName and target
2. httpStatus / SDK error code
3. classification
4. attemptCount
5. elapsedMs
6. finalDisposition (retried_succeeded, retried_failed, no_retry_failed)

Why this matters: your operations team can immediately see if outages are dependency-side transient bursts or persistent code/data issues.

4. DLQ handling
Goal: never silently drop failed work, and avoid infinite poison loops.

1. Move message to DLQ when:
- attempts exhausted
- non-retriable classification
- timeout budget exhausted
2. Include rich DLQ metadata:
- original jobId, correlationId
- dependency classification
- attempt count
- last error summary
- next recommended action (replay_safe, replay_after_fix, manual_review)
3. Build replay policy:
- auto-replay only for transient classes under controlled rate
- manual replay for permanent classes after fix
4. Add dedupe/idempotency key so replay does not create duplicate side effects.

5. Observability and guardrails
Goal: detect degradation before full stoppage.

Emit metrics per dependency:
1. success rate
2. timeout rate
3. retry rate
4. exhausted retry count
5. DLQ enqueue rate
6. p95/p99 latency
7. circuit-open duration (if using circuit breaker)

Set alerts:
1. timeout rate > threshold for 5-10 min
2. DLQ rate > baseline
3. retries per request spike
4. exhausted retries > threshold

How this applies to your case
1. You already saw failed-fast ApiException in EmailQueue path.
2. With this model, that exception gets classified and routed:
- transient -> bounded retries within budget
- permanent -> immediate fail + DLQ with reason
3. If dependency is slow/intermittent, timeout budget prevents worker starvation and fake “hang” symptoms.
4. DLQ gives controlled backlog instead of silent workflow stoppage.

Implementation sequence (low risk first)
1. Add timeout budget + cancellation for outbound API calls.
2. Add retry with jitter for transient classes only.
3. Add classification contract and structured telemetry.
4. Add DLQ decision logic + metadata.
5. Add replay workflow with idempotency guard.
6. Add alerting on timeout/retry/DLQ concentration.

If you want, I can draft a concrete policy matrix next: status/error code to classification, retry decision, and DLQ disposition in one table you can hand to engineering and support.