# Monitoring queries

Two sources, queried in different places:

- **`AzureMetrics`** — platform metrics (CPU, memory, execution units). Routed to our Log
  Analytics workspace by the `cipp-metrics-to-la` diagnostic settings on `cippkvupn`,
  `cippkvupn-proc` and `CIPP-srv-kvupn-premium`. Query from the workspace's **Logs** blade.
- **`requests` / `traces` / `exceptions`** — Application Insights. Query from the App Insights
  resource's **Logs** blade.

Metrics take ~5–15 min to appear. Workspace retention is **30 days**; raw platform metrics are
separately retained 93 days and remain queryable via the Metrics blade.

> Y1 Consumption exposes **no CPU metric**. `cippkvupn-proc` can only be tracked by
> `MemoryWorkingSet`, `FunctionExecutionUnits`, `FunctionExecutionCount`, `Requests`, `Http5xx`
> and `RequestsInApplicationQueue`.

---

## Main app CPU + memory

Watches for the pre-offload failure mode, where memory pinned at ~92 % and stayed there while idle.

```kusto
AzureMetrics
| where TimeGenerated > ago(24h)
| where Resource =~ "CIPP-SRV-KVUPN-PREMIUM"
| where MetricName in ("CpuPercentage", "MemoryPercentage")
| summarize avg = avg(Average), max = max(Maximum)
    by bin(TimeGenerated, 15m), MetricName
| render timechart
```

Memory staying high **while CPU is near zero** is the signature to watch for — that is
accumulation, not work.

## Proc memory against the Y1 ceiling

```kusto
AzureMetrics
| where TimeGenerated > ago(24h)
| where Resource =~ "CIPPKVUPN-PROC"
| where MetricName == "MemoryWorkingSet"
| extend PctOfY1Limit = round(100.0 * Maximum / 1610612736, 1)   // 1.5 GB
| summarize maxMB = round(max(Maximum) / 1048576, 1), maxPct = max(PctOfY1Limit)
    by bin(TimeGenerated, 15m)
| render timechart
```

Sustained >80 % means Consumption is no longer viable for the background load.

## Proc consumption vs the free grant

```kusto
AzureMetrics
| where TimeGenerated > ago(30d)
| where Resource =~ "CIPPKVUPN-PROC"
| where MetricName == "FunctionExecutionUnits"
| summarize MBms = sum(Total)
| extend GBs = round(MBms / 1024 / 1000, 1),
         PctOfFreeGrant = round(100.0 * (MBms / 1024 / 1000) / 400000, 2)
```

Free grant is 400,000 GB-s + 1M executions per subscription per month, shared only with other
**Y1** apps (FlexConsumption bills separately).

## Offload health — main must show ZERO executions

The single most useful query here. Any non-zero value for `CIPPKVUPN` means the main app has
resumed background work, which means offloading has silently broken — almost always version skew.

```kusto
AzureMetrics
| where TimeGenerated > ago(6h)
| where MetricName == "FunctionExecutionCount"
| summarize execs = sum(Total) by bin(TimeGenerated, 30m), Resource
| render timechart
```

## HTTP 5xx across both apps

```kusto
AzureMetrics
| where TimeGenerated > ago(24h)
| where MetricName == "Http5xx" and Total > 0
| summarize errors = sum(Total) by bin(TimeGenerated, 15m), Resource
| render columnchart
```

---

# Application Insights

## Request latency vs the 45-second SWA limit

The most important application query. Anything over 45 s is returned to the user as a **500 by
the SWA gateway**, even though it is recorded here as a success.

```kusto
requests
| where timestamp > ago(24h) and name == "CIPPHttpTrigger"
| summarize reqs = count(),
            p50 = round(percentile(duration, 50) / 1000, 1),
            p95 = round(percentile(duration, 95) / 1000, 1),
            over45s = countif(duration > 45000)
    by bin(timestamp, 1h)
| render timechart
```

Reference: 2026-09-11 pre-offload — p50 39.6 s, p95 118 s, 44 % over 45 s.

## Slowest endpoints

```kusto
requests
| where timestamp > ago(24h) and name == "CIPPHttpTrigger"
| extend path = tostring(parse_url(url).Path)
| summarize n = count(),
            p95 = round(percentile(duration, 95) / 1000, 1),
            mx = round(max(duration) / 1000, 1)
    by path
| order by p95 desc
| take 25
```

## Runspace pool health

Each `#### CIPP-API Start ####` is one runspace initialising — roughly 9 s of CPU. A burst of
these after a quiet period means the pool was emptied, which normally indicates a restart or
deploy rather than idle decay.

```kusto
traces
| where timestamp > ago(24h) and cloud_RoleName == "cippkvupn"
| where message has "#### CIPP-API Start ####"
| summarize inits = count() by bin(timestamp, 15m)
| render columnchart
```

## Worker-channel aborts

When one function hits `functionTimeout`, the PowerShell worker restarts and **aborts every
other in-flight invocation on that worker** — producing correlated 500s across unrelated
endpoints. Was ~40/hour before offloading; should stay at 0.

```kusto
exceptions
| where timestamp > ago(24h)
| where outerMessage has_any ("Worker channel is shutting down", "Timeout value")
| summarize aborts = count() by bin(timestamp, 1h), cloud_RoleName
```

## Which node ran which timer

Confirms work is landing on `cippkvupn-proc` and not the main app.

```kusto
traces
| where timestamp > ago(6h)
| where message has "CIPPTimer:"
| project timestamp, cloud_RoleName, message
| order by timestamp desc
```

## Trace one slow request end to end

```kusto
let opId = "<operation_Id>";
union requests, traces, exceptions, dependencies
| where operation_Id == opId
| project timestamp, itemType, message = coalesce(message, name, outerMessage), duration
| order by timestamp asc
```
