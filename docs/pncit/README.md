# PNC IT — CIPP deployment notes

Operational notes for our self-hosted CIPP instance. This directory is **ours**, not upstream —
nothing here is synced from `KelvinTegelaar/CIPP-API`, so it should never conflict on merge.

> This repository is public. Do not put subscription IDs, tenant IDs, object IDs, keys,
> connection strings, or security posture details in here.

---

## Topology

```
                  pncit/CIPP (main)                pncit/CIPP-API (master)
                         |                                   |
              Azure Static Web Apps CI/CD          Build and deploy Powershell
                         |                                   |
                         v                         +---------+---------+
                 cipp-swa-kvupn                    |                   |
                 (SWA, Standard)                   v                   v
                         |                    cippkvupn          cippkvupn-proc
                         |  /api proxy        (EP1, HTTP only)   (Y1, all background)
                         +------------------->      |                   |
                                                    +-------+-----------+
                                                            |
                                        cippstgkvupn (tables/queues/durable state)
                                        cippkvupn    (Key Vault — secrets)
                                        cippkvupn    (App Insights)
```

| Resource | Kind | Role |
| --- | --- | --- |
| `cipp-swa-kvupn` | Static Web App (Standard) | Frontend; proxies `/api` to `cippkvupn` via linked backend |
| `cippkvupn` | Function App on **EP1** (1 vCPU / 3.5 GiB) | **HTTP only.** Serves the UI API |
| `cippkvupn-proc` | Function App on **Y1 Consumption** | **All background work** — timers, orchestrators, activities, queues |
| `CIPP-srv-kvupn-premium` | App Service plan | EP1 plan behind `cippkvupn` |
| `cipp-srv-kvupnproc` | App Service plan | Y1 Dynamic plan behind `cippkvupn-proc` |
| `cippstgkvupn` | Storage account | Shared by both apps. Separate durable task hubs |
| `cippkvupn` | Key Vault | Both apps' managed identities have `secrets: all` |

---

## Function offloading

We run CIPP's [function offloading](https://docs.cipp.app/user-documentation/cipp/advanced/super-admin/function-offloading)
with a single `proc` node. Around 50 tenants × 31 background timers was enough load to starve
the UI when everything ran on one app.

**Enabled via:** `Config` table → `PartitionKey`/`RowKey` = `OffloadFunctions`, `state = true`
(must be `Edm.Boolean`, not a string). Also togglable in CIPP → Advanced → SuperAdmin.

When enabled, `Set-CIPPOffloadFunctionTriggers` automatically writes these to the **main** app,
which restarts it:

```
AzureWebJobs.CIPPTimer.Disabled            = 1
AzureWebJobs.CIPPActivityFunction.Disabled = 1
AzureWebJobs.CIPPOrchestrator.Disabled     = 1
AzureWebJobs.CIPPQueueTrigger.Disabled     = 1
```

So the main app keeps only `CIPPHttpTrigger`. That is intended.

### `proc` is the catch-all

Valid suffixes are fixed in `Get-CippOffloadSuffix.ps1`: `proc`, `auditlog`, `standards`,
`usertasks`. Apps must be named exactly `<mainapp>-<suffix>`.

Of 31 timers in `Config/CIPPTimers.json`, **29 have `RunOnProcessor: true`** and move to the
offload node. Only 5 name a `PreferredProcessor`; when that node isn't deployed they fall
through to `proc`. So one `proc` app absorbs everything.

**Never deploy a named lane without `proc`.** Timers with no `PreferredProcessor` are skipped
on any node not in `('http','proc')`, and the main app has stepped down — so they would
silently stop running everywhere.

### Known gap

Two timers have `RunOnProcessor: false` (`Start-ContainerUpdateCheck`, `Start-UserSyncTimer`).
With offloading on, the main app's `CIPPTimer` is disabled, and the proc node only runs
`RunOnProcessor: true`. These appear not to run anywhere. Upstream behaviour, not ours.

---

## ⚠️ Version lockstep is mandatory

`Get-CIPPTimerFunctions.ps1`:

```powershell
$AvailableNodes = $Nodes | Where-Object {
    (Test-CippOffloadFunctionApp -SiteName $_.RowKey) -and $_.Version -eq $MainFunctionVersion
}
```

If `cippkvupn-proc`'s version does **not** exactly match `cippkvupn`'s, the node is silently
dropped from `$AvailableNodes` — no error, no log. The main app then reverts to
`$RunOnProcessor = $true` and runs the timers itself, **while the proc node (which has
`CIPP_PROCESSOR=true`) also still runs them**. Both apps double-dispatch every background job
against every tenant.

This is why `master_cippkvupn.yml` deploys the same checkout to both apps in one run.
Do not split them into separate workflows or separate triggers.

Versions are registered in the `Version` storage table, one row per node. The `frontend` row
is explicitly excluded from `$Nodes` and is irrelevant to offloading.

---

## Constraints worth remembering

| Constraint | Value | Consequence |
| --- | --- | --- |
| **SWA proxy timeout** | **45 s** | Any API call slower than this returns **500 from the gateway** even though the function later completes with 200. App Insights will show success while users see errors |
| **Y1 Consumption memory** | **1.5 GB / instance** | Hard ceiling for `cippkvupn-proc`. No CPU metric exists on Y1 |
| **Y1 free grant** | 400,000 GB-s + 1M executions / month, per subscription | Shared only with other *Y1* apps. FlexConsumption apps bill separately |
| **EasyAuth on `cippkvupn`** | `requireAuthentication: true`, no excluded paths | Unauthenticated requests are rejected by middleware and **never reach the function**. A naive URL ping cannot warm the app |
| **`functionTimeout`** | `00:10:00` (`host.json`) | Also the Y1 maximum |
| **PowerShell worker concurrency** | default 1000 runspaces (Functions v4) | Each CIPP runspace costs roughly 250–300 MB and ~9 s of `profile.ps1`. We cap it explicitly on both apps |
| **Frontend build** | Node from `engines.node` | Oryx cannot build it; we build on the runner and use `skip_app_build`. See `pncit/CIPP` workflow |

---

## Runbook

### Is offloading actually working?

1. **Main app should run zero function executions.** Any `FunctionExecutionCount` on `cippkvupn`
   means it has resumed background work — i.e. offloading broke.
2. `Version` table: rows for `cippkvupn` and `cippkvupn-proc` must match exactly.
3. Main app should have the four `AzureWebJobs.*.Disabled = 1` settings.
4. `CIPPTimers` table `LastOccurrence` should be advancing.

See `monitoring-queries.md`.

### Symptom → cause

| Symptom | Look at |
| --- | --- |
| Login stalls, intermittent 500s | Request duration vs the 45 s SWA limit. Check memory on the EP1 plan |
| Duplicated/doubled background jobs | Version skew between the two apps |
| Background jobs stopped entirely | Offload toggle on but no healthy `proc` node |
| Slow first load after a deploy | Expected — a restart empties the runspace pool; ~9 s per runspace to rebuild |
| Correlated 500s across unrelated endpoints | `Worker channel is shutting down` — one timeout kills every in-flight invocation on that worker |

### Rolling offloading back

1. `Config` table → `OffloadFunctions.state = false`
2. Remove the four `AzureWebJobs.*.Disabled` settings from `cippkvupn` (or wait for
   `Set-CIPPOffloadFunctionTriggers` to clear them)
3. Optionally stop `cippkvupn-proc`

The main app resumes all timers once no healthy offload node is visible.

---

## Reference points (2026-09-11)

Captured before and after enabling offloading, same instance, same day.

| Metric | Before | After |
| --- | --- | --- |
| `CIPPHttpTrigger` p50 | 39.6 s | 0.1–0.7 s warm |
| `CIPPHttpTrigger` p95 | 118 s | — |
| Requests over the 45 s SWA limit | 44 % | 0 |
| EP1 memory while idle | pinned at 92 % | ~35 %, flat |
| `Worker channel is shutting down` aborts | ~40 / hour | 0 |
| Background timers on main | 29 | 0 |
| `cippkvupn-proc` memory | — | ~717 MB (~47 % of the Y1 ceiling) |

The pre-change 92 % memory figure was **not** a leak. It was ~50 tenants' worth of background
orchestration accumulating runspaces on a 3.5 GiB instance that also had to serve the UI.

The runspace pool survives at least ~35 minutes idle without re-initialising, so routine idle
does not cause slow logins. Restarts and deploys do.
