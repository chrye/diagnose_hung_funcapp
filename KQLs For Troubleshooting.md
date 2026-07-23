Use these as a ready-to-run KQL pack in Application Insights Logs.

### How to use these together quickly

1. Run Query 2 first to identify slow operations.
2. Run Query 3 to see which dependency targets/types are failing or timing out.
3. Run Query 1 to detect true stalls (started but no terminal event beyond threshold).

#### Query 1: Started operations with no success/failure within threshold (Function-host trace based)

```
let lookback = 24h;
let stallThreshold = 15m;
let started =
    traces
    | where timestamp > ago(lookback)
    | where message startswith "Executing '"
    | extend invocationId = extract(@"Id=([0-9a-fA-F-]{36})", 1, message)
    | extend functionName = extract(@"Executing '([^']+)'", 1, message)
    | where isnotempty(invocationId)
    | project invocationId, functionName, startTime = timestamp;
let terminal =
    traces
    | where timestamp > ago(lookback)
    | where message startswith "Executed '"
    | extend invocationId = extract(@"Id=([0-9a-fA-F-]{36})", 1, message)
    | extend terminalState = extract(@"\(([^,]+), Id=", 1, message)   // Succeeded / Failed
    | where isnotempty(invocationId)
    | summarize endTime = max(timestamp), terminalState = any(terminalState) by invocationId;
started
| join kind=leftouter terminal on invocationId
| extend age = coalesce(endTime, now()) - startTime
| where isnull(endTime) and age > stallThreshold
| order by age desc
| project startTime, age, functionName, invocationId

```

If you already emit custom stage events, this is even stronger (replace event names with yours):

```
let lookback = 24h;
let stallThreshold = 20m;
let started =
    customEvents
    | where timestamp > ago(lookback)
    | where name in ("workflow_started", "job_started")
    | project opId = operation_Id, jobId = tostring(customDimensions.jobId), startTime = timestamp;
let terminal =
    customEvents
    | where timestamp > ago(lookback)
    | where name in ("workflow_completed", "workflow_failed", "workflow_stalled_timeout")
    | summarize endTime = max(timestamp), terminalEvent = arg_max(timestamp, name) by opId = operation_Id;
started
| join kind=leftouter terminal on opId
| extend age = coalesce(endTime, now()) - startTime
| where isnull(endTime) and age > stallThreshold
| order by age desc
```

#### Query 2: p95/p99 duration and count exceeding threshold

```
let lookback = 24h;
let durationThreshold = 30s;
requests
| where timestamp > ago(lookback)
| summarize
    total = count(),
    overThreshold = countif(duration > durationThreshold),
    overThresholdPct = 100.0 * countif(duration > durationThreshold) / count(),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)
  by operationName = name
| order by p99 desc
```

Optional hourly trend view:

```
let lookback = 24h;
let durationThreshold = 30s;
requests
| where timestamp > ago(lookback)
| summarize
    total = count(),
    overThreshold = countif(duration > durationThreshold),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)
  by bin(timestamp, 1h), operationName = name
| order by timestamp asc
```

#### Query 3: Dependency failures by target/type to find trigger systems

Basic failure hotspot view:
```
let lookback = 24h;
dependencies
| where timestamp > ago(lookback)
| summarize
    total = count(),
    failed = countif(success == false),
    failRatePct = 100.0 * countif(success == false) / count(),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99),
    resultCodes = make_set(resultCode, 10)
  by depType = type, target, depName = name
| where failed > 0
| order by failed desc, failRatePct desc
```

Enriched with exception type/message (best for root-cause signal):
```
let lookback = 24h;
let depFailures =
    dependencies
    | where timestamp > ago(lookback)
    | where success == false
    | project operation_Id, depType = type, target, depName = name, resultCode, depDuration = duration, depTime = timestamp;
depFailures
| join kind=leftouter (
    exceptions
    | where timestamp > ago(lookback)
    | project operation_Id, exType = type, exMessage = outerMessage
  ) on operation_Id
| summarize
    failed = count(),
    p95 = percentile(depDuration, 95),
    p99 = percentile(depDuration, 99),
    resultCodes = make_set(resultCode, 10),
    exTypes = make_set(exType, 10),
    sampleMessages = make_set(exMessage, 5)
  by depType, target, depName
| order by failed desc
```

#### companion query: Explicit start-to-failure latency distribution for EmailQueue failures

```
let lookback = 24h;
requests
| where timestamp > ago(lookback)
| where name == "EmailQueue" or operation_Name == "EmailQueue"
| where success == false
| summarize failures=count() by bin(duration, 1s)
| order by duration asc
```

#### companion query: Explicit oldest in-flight age single-value summary

```
let lookback = 24h;
let stallThreshold = 20m;
let fn = "EmailQueue";
let started =
    traces
    | where timestamp > ago(lookback)
    | where message startswith strcat("Executing '", fn, "'")
    | extend invocationId = extract(@"Id=([0-9a-fA-F-]{36})", 1, message)
    | where isnotempty(invocationId)
    | project invocationId, startTime = timestamp;
let terminal =
    traces
    | where timestamp > ago(lookback)
    | where message startswith strcat("Executed '", fn, "'")
    | extend invocationId = extract(@"Id=([0-9a-fA-F-]{36})", 1, message)
    | where isnotempty(invocationId)
    | summarize endTime=max(timestamp) by invocationId;
started
| join kind=leftouter terminal on invocationId
| where isnull(endTime)
| extend age = now() - startTime
| where age > stallThreshold
| summarize oldestInFlight=max(age), inFlightCount=count()
```

#### companion query: timeout-focused KQL query specifically for Service Bus dependencies, using both result code and message/exception timeout patterns.

```
let lookback = 24h;
let timeoutDuration = 30s;
let timeoutRegex = @"(?i)(timeout|timed out|operationtimedout|servicetimeout|requesttimedout|gateway timeout|504|408)";
let sbDeps =
    dependencies
    | where timestamp > ago(lookback)
    | where type has "Service Bus"
        or target has "servicebus.windows.net"
        or name has_cs "ServiceBus"
    | project
        operation_Id,
        depTime = timestamp,
        depName = name,
        depType = type,
        depTarget = target,
        depSuccess = success,
        depDuration = duration,
        depResultCode = tostring(resultCode);
let exByOp =
    exceptions
    | where timestamp > ago(lookback)
    | summarize
        exTypes = make_set(type, 10),
        exMessages = make_set(outerMessage, 10)
      by operation_Id;
let traceByOp =
    traces
    | where timestamp > ago(lookback)
    | summarize
        traceMessages = make_set(message, 10)
      by operation_Id;
sbDeps
| join kind=leftouter exByOp on operation_Id
| join kind=leftouter traceByOp on operation_Id
| extend timeoutByCode =
    depResultCode matches regex timeoutRegex
| extend timeoutByException =
    tostring(exMessages) matches regex timeoutRegex
| extend timeoutByTrace =
    tostring(traceMessages) matches regex timeoutRegex
| extend timeoutByDuration =
    depDuration >= timeoutDuration and depSuccess == false
| extend isTimeout =
    timeoutByCode or timeoutByException or timeoutByTrace or timeoutByDuration
| summarize
    totalCalls = count(),
    failedCalls = countif(depSuccess == false),
    timeoutCalls = countif(isTimeout),
    timeoutRatePct = 100.0 * countif(isTimeout) / count(),
    failRatePct = 100.0 * countif(depSuccess == false) / count(),
    p95Duration = percentile(depDuration, 95),
    p99Duration = percentile(depDuration, 99),
    timeoutResultCodes = make_set_if(depResultCode, isTimeout, 10),
    timeoutExceptionTypes = make_set_if(tostring(exTypes), isTimeout, 10),
    timeoutSamples = make_set_if(strcat("code=", depResultCode, "; msg=", substring(tostring(exMessages), 0, 180)), isTimeout, 5)
  by depType, depTarget, depName
| where timeoutCalls > 0
| order by timeoutCalls desc, timeoutRatePct desc
```

#### companion query: Optional drill-down (raw timed-out operations to inspect one-by-one)

```
let lookback = 24h;
let timeoutDuration = 30s;
let timeoutRegex = @"(?i)(timeout|timed out|operationtimedout|servicetimeout|requesttimedout|gateway timeout|504|408)";
let sbDeps =
    dependencies
    | where timestamp > ago(lookback)
    | where type has "Service Bus"
        or target has "servicebus.windows.net"
        or name has_cs "ServiceBus"
    | project operation_Id, timestamp, name, type, target, success, duration, resultCode=tostring(resultCode);
sbDeps
| join kind=leftouter (
    exceptions
    | where timestamp > ago(lookback)
    | summarize exType=any(type), exMessage=any(outerMessage) by operation_Id
  ) on operation_Id
| extend isTimeout =
    resultCode matches regex timeoutRegex
    or tostring(exMessage) matches regex timeoutRegex
    or (duration >= timeoutDuration and success == false)
| where isTimeout
| order by timestamp desc
| project timestamp, operation_Id, name, type, target, success, duration, resultCode, exType, exMessage
```



### Here is a version filtered specifically to EmailQueue and Service Bus paths based on your current incident pattern, filtered to `EmailQueue` and Service Bus.

Query 1: `EmailQueue` starts with no terminal state beyond threshold (possible stall)

```kusto
let lookback = 24h;
let stallThreshold = 20m;
let fn = "EmailQueue";
let started =
    traces
    | where timestamp > ago(lookback)
    | where message startswith strcat("Executing '", fn, "'")
    | extend invocationId = extract(@"Id=([0-9a-fA-F-]{36})", 1, message)
    | where isnotempty(invocationId)
    | project invocationId, startTime = timestamp;
let terminal =
    traces
    | where timestamp > ago(lookback)
    | where message startswith strcat("Executed '", fn, "'")
    | extend invocationId = extract(@"Id=([0-9a-fA-F-]{36})", 1, message)
    | extend terminalState = extract(@"\(([^,]+), Id=", 1, message)   // Succeeded / Failed
    | where isnotempty(invocationId)
    | summarize endTime = max(timestamp), terminalState = any(terminalState) by invocationId;
started
| join kind=leftouter terminal on invocationId
| extend age = coalesce(endTime, now()) - startTime
| where isnull(endTime) and age > stallThreshold
| order by age desc
| project startTime, age, invocationId
```

Query 2: `EmailQueue` p95/p99 and over-threshold counts

```kusto
let lookback = 24h;
let durationThreshold = 30s;
requests
| where timestamp > ago(lookback)
| where name == "EmailQueue" or operation_Name == "EmailQueue"
| summarize
    total = count(),
    failed = countif(success == false),
    overThreshold = countif(duration > durationThreshold),
    overThresholdPct = 100.0 * countif(duration > durationThreshold) / count(),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)
```

Query 3: Service Bus dependency failures only (target/type hotspots)

```kusto
let lookback = 24h;
dependencies
| where timestamp > ago(lookback)
| where type has "Azure Service Bus"
   or target has "servicebus.windows.net"
   or name has_cs "ServiceBus"
| summarize
    total = count(),
    failed = countif(success == false),
    failRatePct = 100.0 * countif(success == false) / count(),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99),
    resultCodes = make_set(resultCode, 10)
  by depType = type, target, depName = name
| where failed > 0
| order by failed desc, failRatePct desc
```

Query 4: Correlate `EmailQueue` failures to Service Bus + exception types (best incident view)

```kusto
let lookback = 24h;
let emailReq =
    requests
    | where timestamp > ago(lookback)
    | where name == "EmailQueue" or operation_Name == "EmailQueue"
    | project operation_Id, reqTime=timestamp, reqDuration=duration, reqSuccess=success;
let sbDeps =
    dependencies
    | where timestamp > ago(lookback)
    | where type has "Azure Service Bus"
       or target has "servicebus.windows.net"
       or name has_cs "ServiceBus"
    | project operation_Id, depTime=timestamp, depName=name, depTarget=target, depType=type, depSuccess=success, depDuration=duration, depCode=resultCode;
let ex =
    exceptions
    | where timestamp > ago(lookback)
    | project operation_Id, exType=type, exMessage=outerMessage;
emailReq
| join kind=leftouter sbDeps on operation_Id
| join kind=leftouter ex on operation_Id
| summarize
    requests=count(),
    requestFailures=countif(reqSuccess == false),
    sbCalls=countif(isnotempty(depName)),
    sbFailures=countif(depSuccess == false),
    p95Req=percentile(reqDuration, 95),
    p99Req=percentile(reqDuration, 99),
    p95Sb=percentile(depDuration, 95),
    p99Sb=percentile(depDuration, 99),
    sbTargets=make_set(depTarget, 10),
    sbCodes=make_set(depCode, 10),
    exTypes=make_set(exType, 10),
    exSamples=make_set(exMessage, 5)
```

Query 5: Detect the exact incident signature text you saw (`Message processing error`, `Action=UserCallback`)

```kusto
let lookback = 24h;
traces
| where timestamp > ago(lookback)
| where message has "Message processing error"
   and message has "Action=UserCallback"
   and message has "servicebus.windows.net"
| project timestamp, severityLevel, operation_Id, message
| order by timestamp desc
```

Tip for your environment: if `requests.name` does not equal `EmailQueue`, swap the filter to `operation_Name has "EmailQueue"` in all queries.




