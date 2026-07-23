## Technical recommendations (short-term and medium-term)

### Short-Term Hardening Recommendations

#### Immediate hardening actions (implement now)

1. Replace blocking task waits with async coordination.
Replace Task.WaitAll and task.Result usage with await Task.WhenAll in the plugin initialization paths to avoid blocked worker threads and improve cancellation behavior.
[WorkflowManager.Core/Services/PluginRunner.cs]

2. Remove unbounded completion loops.
Replace any open-ended list-count waiting loops with explicit completion validation and bounded timeout logic, so initialization cannot wait indefinitely.
[WorkflowManager.Core/Services/PluginRunner.cs]

3. Return explicit initialization outcomes.
Update init methods to return structured results (for example: Success, AddedExecution, SequenceId, FailureReason) rather than inferring correctness from shared mutable collections.

4. Fix ProvisionSb failure-state propagation.
Correct event/state capture so job creation failure is reliably surfaced to the caller and downstream handling logic. [ProvisionSb.cs]

5. Apply timeout budgets and cancellation tokens to dependency calls.
Add per-call timeout and cancellation to Key Vault, API metadata, DB, and related init/dependency operations, with an overall step-level time budget. [PluginRunner.cs]

6. Add deterministic runner lifecycle cleanup.
Ensure runner entries are removed on terminal states, event handlers are unsubscribed, and active-runner count telemetry is emitted to prevent long-run accumulation. [WorkflowManager.Core/Services/WorkFlowManagerNew.cs]

7. Remove untracked fire-and-forget persistence.
Replace Task.Factory.StartNew-style persistence calls [Task.Factory.StartNew for SaveDataToDb appears in many places, for example PluginRunner.cs] with awaited async persistence or reliable queued persistence to avoid silent failures and orphaned work.

8. Keep weekly restart only as temporary mitigation.
Continue weekend restart only as an operational guardrail until hardening changes are deployed and validated; do not treat restart as the fix.

#### Diagnostic resilience actions (implement in parallel)

9. Strengthen observability for failure-mode separation.
With sampling disabled and exceptions now visible, add workflow-stage telemetry so we can clearly distinguish failed-fast errors from true hangs/stalls.

10. Add a stuck-workflow detector.
Implement per-stage timeout plus heartbeat/state-transition logging for each workflow job to identify exactly where long-running jobs stop progressing.

11. Add outbound dependency resilience.
Implement bounded retry policy, timeout budget, failure classification, and DLQ/poison handling for outbound API paths. This is especially important given the 7/14 Service Bus processing path showing downstream ApiException behavior.

Expected outcome
1. Reduced recurrence risk from known code-path hang/stall patterns.
2. Faster, evidence-based isolation of root cause if incidents recur.
3. Clear separation of transient dependency failures vs workflow orchestration stalls.
4. Ability to retire weekly restart once stability is demonstrated over a full operating window.


#### **Patch Goal**
Keep Raja's sequencing intent (barrier before processing) while removing hang-prone patterns in the active .NET isolated workflow path.

**Scope**
- PluginRunner.cs
- ProvisionSb.cs
- WorkFlowManagerNew.cs

---

**Phase 1: Fix Immediate Hang Risks in Plugin Initialization**

1. Replace blocking barriers in runner init methods.
Edit these methods in PluginRunner.cs:
- `InitializeRunner`
- `ReInitializeRunner`
- `ReInitializeRunnerApproval`
- `ReInitializeRunnerRestart`

Current pattern:
- `Task.Run(() => InitalizeApis(...))`
- `Task.WaitAll(...)`
- `tasks.Any(x => !x.Result)`
- `while (ApiExecutionList.Count != tasks.Count) { await Task.Delay(50); }`

Replace with:
- `var results = await Task.WhenAll(...)`
- no `Task.WaitAll`
- no `.Result`
- remove count-based infinite wait loop

2. Return explicit per-init outcomes.
In PluginRunner.cs, add a private result type, for example:
- `Success`
- `AddedExecution`
- `SequenceId`
- `ErrorMessage`

Update `InitalizeApis` and `ReInitalizeApis` to return this result instead of only `bool`.

Why:
- Current code can "succeed" without adding execution entries, then wait forever on list count matching.

3. Fail fast on "success but no execution item".
After `WhenAll`, validate:
- any failed result -> `OnPluginsCreateFailed(...)` and return
- any result with `AddedExecution == false` -> fail with clear message and return
- only proceed to `PerformOperations` if at least one valid execution item exists

4. Add timeout and cancellation around init.
Use a `CancellationTokenSource` with max duration in init phase.
If timeout hits, fail job creation explicitly and log sequence IDs still pending.

---

**Phase 2: Fix Confirmed ProvisionSb Failure-State Bug**

1. Fix pass-by-value event state bug.
In ProvisionSb.cs, the handler writes to a local argument, not the outer variable.

Current behavior:
- `jobEventArgs` remains null
- failure path can be missed

Patch options:
- Use `TaskCompletionSource<JobEventArgs>` with timeout
- or capture into outer variable directly in lambda

Recommended:
- Replace `AutoResetEvent + local JobEventArgs` with `TaskCompletionSource<JobEventArgs>`
- `await Task.WhenAny(tcs.Task, Task.Delay(MaxWait))`

2. Unsubscribe event handler after completion.
In ProvisionSb.cs, always detach `JobsCreateStatus` handler in `finally`.

---

**Phase 3: Add Lifecycle Cleanup for Singleton Runner Store**

1. Track and remove finished runners.
In WorkFlowManagerNew.cs, `_pluginRunners` is populated but not removed.

Add:
- terminal callback/hook from runner completion/failure
- `_pluginRunners.TryRemove(jobId, out _)` on terminal states
- event unsubscription for runner events at cleanup time

2. Add minimal telemetry counters.
Log:
- runner count on add/remove
- active job count
- job age for long-running jobs

This gives immediate evidence if weekly restart is clearing accumulation.

---

**Phase 4: Remove Fire-and-Forget DB Save Calls in Runner**

1. Replace `Task.Factory.StartNew(() => SaveDataToDb(...))`.
Occurrences in PluginRunner.cs should move to awaited async calls or queued persistence with explicit error handling.

2. If immediate await is too invasive:
- add centralized bounded queue/worker for persistence
- capture and log all exceptions
- never drop save tasks silently

---

**Phase 5: Keep Current Architecture, But Make It Observable**

1. Correlation logs in every critical path.
Ensure all logs include:
- `JobId`
- `CorrelationId`
- `Sequence`
- `ApiId`
- `Stage` (`init`, `execute`, `ack`, `response`, `complete`, `fail`)

2. Add "stuck guard" status transitions.
If step hasn't progressed within threshold, mark job as stalled with explicit reason.

---

**Phase 6: Medium-Term Durable Migration (Separate Workstream)**

1. Keep this as separate epic, not blocking hotfix.
2. Preserve current business semantics:
- retry/restart behavior
- per-sequence processing
- approval checkpoints
3. Move orchestration state from in-memory collections to durable orchestration history.

#### Practical Incident runbook

1. First 10 minutes in incident
- Check failed requests/exceptions in the incident window.
- If many failures are under, say, 5-10s with clear exception types, classify as failed-fast dominant.
- If failures are low but business reports stuck jobs, suspect stall.

2. KQL confirmation (See: KQLs For Troubleshooting.md)
- Query started operations with no success/failure event within threshold.
- Query p95/p99 duration and count of requests exceeding threshold.
- Query dependency failures by target/type to find trigger systems.

3. Decide action path
- Failed-fast dominant: Fix error handling, retries, timeout budgets, fallback policy, poison/dead-letter behavior.
- Stall dominant: Fix orchestration lifecycle: bounded waits, cancellation flow, in-memory state cleanup, no fire-and-forget critical paths, watchdog transitions.

#### What to implement in code so KQL becomes definitive

1. Emit workflow stage events:
- received, initialized, dependency_call_start, dependency_call_end, persisted, completed, failed, stalled_timeout
2. Include dimensions on every event:
- jobId, correlationId, functionName, stage, dependencyName, elapsedMs, outcome, exceptionType
3. Add hard timeout checkpoints:
- If stage exceeds threshold, emit stalled_timeout and move to deterministic failure state.
4. Add runner lifecycle metrics:
- active runner count, oldest runner age, cleanup success/failure.

### Medium-term architecture recommendation

1. Move full production workflow orchestration to Durable Functions (or equivalent durable orchestration).
Transcript confirms this is already in progress.
See transcript.

2. Use incident journal + App Insights map + KQL correlation at each recurrence to validate whether stalls are dependency-triggered and non-recovering due to code path.
Discussion reflected at transcript.

3. Priority adjustment: if another stoppage occurs after short-term hardening, escalate durable migration timeline immediately.
