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
