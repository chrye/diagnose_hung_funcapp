What "stuck workflow detector" means
1. Treat each workflow job as a state machine.
2. Emit a telemetry event every time the job changes state.
3. Track last-progress time per job.
4. If a state exceeds its allowed time budget, emit a timeout/stall event with the exact state name and context.

This gives you evidence like:
1. Job started at 02:01:03.
2. Reached state DependencyCall:KeyVault at 02:01:05.
3. No further state change for 10 minutes.
4. Detector emitted stalled_timeout at state DependencyCall:KeyVault.

Why this is needed in your case
1. Current symptom includes both fast failures (7/14 EmailQueue ApiException) and periodic "stops progressing" behavior.
2. Request-level telemetry alone can miss where inside the workflow it stalled.
3. State-transition telemetry tells you if the job died quickly, retried forever, or got stuck at a specific dependency boundary.

Recommended design (practical, low overhead)
1. Define a canonical state list
- received
- validated
- persisted_job
- enqueue_next_step
- dependency_call_start
- dependency_call_end
- business_step_completed
- completed
- failed
- stalled_timeout

2. Emit one structured event per transition
Include these fields every time:
1. jobId
2. correlationId
3. functionName
4. stage (state name)
5. stageSequence (monotonic int)
6. timestampUtc
7. elapsedMsSinceJobStart
8. dependencyName (if applicable)
9. attempt
10. outcome
11. exceptionType
12. exceptionMessageShort

3. Per-stage timeout policy (SLO budget)
Create config like:
1. dependency_call_start -> dependency_call_end: 60s
2. enqueue_next_step: 15s
3. business_step_completed: 120s
4. total job age cap: 20m

4. Heartbeat policy
1. While in long-running state, emit heartbeat every 30-60s.
2. Heartbeat must include:
- currentStage
- ageInStage
- retryCount
- hostInstanceId
3. If no heartbeat and no transition within threshold, classify as suspected hard stall.

5. Detector logic
1. Real-time: background monitor checks active jobs every minute.
2. Offline: KQL query computes stale jobs (no transition past threshold).
3. When breached:
- emit stalled_timeout event
- include lastKnownStage, ageInStage, dependency target, operation id
- optionally mark job status in DB as Stalled for operational visibility

How to implement in code (minimal intrusion)
1. Wrap stage execution in a helper
- EnterStage(stageName)
- ExitStage(stageName, outcome)
- FailStage(stageName, exception)
2. Use a central telemetry writer so every event has consistent dimensions.
3. Store active job runtime state in memory plus persistent status row (for cross-restart continuity).
4. Add cancellation token and timeout wrappers around dependency calls.
5. Ensure fire-and-forget paths are either removed or instrumented with independent completion events.

What this enables in KQL immediately
1. Exact "where stuck" query
- last stage per job + age in stage
2. Timeout concentration by stage
- which stage breaches most often
3. Dependency linkage
- stalled jobs grouped by dependencyName/target
4. Distinguish cleanly:
- failed-fast: terminal failed event soon after start
- stall: no terminal event, stage age exceeds threshold, heartbeat gap

Concrete telemetry event examples
1. workflow_state_transition
- stage: dependency_call_start
- dependencyName: servicebus_send
- stageSequence: 5

2. workflow_heartbeat
- currentStage: dependency_call_start
- ageInStageMs: 45000

3. workflow_stalled_timeout
- currentStage: dependency_call_start
- ageInStageMs: 600000
- timeoutBudgetMs: 60000
- dependencyName: servicebus_send

Operational rollout plan
1. Week 1
- Add transition events + stageSequence + required dimensions.
2. Week 2
- Add per-stage timeout config + heartbeat.
3. Week 2
- Add detector event stalled_timeout and dashboard tiles.
4. Week 3
- Tune thresholds from observed p95/p99 per stage.

Success criteria
1. Every stuck incident yields a lastKnownStage and ageInStage within 5 minutes.
2. Unknown "hung somewhere" incidents drop to near zero.
3. You can rank top 3 problematic stages/dependencies from telemetry alone.

DevOps' Christmas Present: A KQL dashboard (stale jobs, stage timeout heatmap, dependency-linked stalls).

