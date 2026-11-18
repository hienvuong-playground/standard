# 0019 – Stop AKS and Postgres every night, start them by hand

## Context

The playground is only used during work sessions, but AKS nodes and the Postgres flexible server bill for compute around the clock. Work hours are irregular, so there is no fixed "start" time, but there is a time by which work is surely over.

## Decision

**Stop on a schedule, start by hand.**

- Stop: Azure Automation, `infra/cloud/runbook.tf` with scripts in `infra/cloud/scripts/`. One daily schedule at 02:00 Vietnam time runs two runbooks, `Stop-AKS-Cluster` and `Stop-PostgreSQL-Server`. Both skip if the resource isn't running. The Automation account's system-assigned identity has `Contributor` scoped to the cluster and to the server only, not the resource group.
- Start: `infra/start-cloud.sh` starts both in parallel and skips whatever is already running. No start schedule, because there is no fixed start time.

**Runtime: PowerShell 7.6 runtime environment.** The runbooks first ran on PowerShell 7.2. Azure Automation retired 7.1/7.2 on 2026-09-30. On 7.2, `Get-AzPostgreSqlFlexibleServer` was "not recognized" even though `Az.PostgreSql` 1.1.0 was listed in the account's 7.2 modules. The cause wasn't found, and it wasn't worth digging into on a retired runtime. Microsoft says to use a runtime environment with 7.6 or 7.4. 7.4 ends support 2026-11-10, so 7.6 it is (LTS until 2028-11-14). Its default Az 15.1.0 includes `Az.PostgreSql` 1.4.0, so nothing extra is imported. With runtime environments, `runbook_type` is plain `PowerShell` and matches what the API returns. That removed the old `ignore_changes = [runbook_type]` workaround.

Rejected:
- Calling ARM directly with `Invoke-AzRestMethod`. It works, but it hides a runtime problem instead of fixing it.
- Importing modules into the 7.2 runtime, because 7.2 is retired.
- Postgres "automation tasks" (portal → server → Tasks). These are Logic Apps created in the portal, need an interactive sign-in for the connection, and would sit apart from the AKS runbook.

**Schedule details.** azurerm ignores `start_time` after create, so the stop time is part of the schedule name (`stop-aks-daily-0200`). Changing `local.aks_stop_time` recreates the schedule and its job links. `start_time` is the UTC date + 48h at the stop time, so a recreate always lands in the future (Azure rejects less than 5 minutes ahead).

**Scripts fail loudly.** Both set `$ErrorActionPreference = "Stop"`. Without it, the "not recognized" error above was non-terminating: the script carried on, saw an empty state, printed "Skipping stop", and the job showed `Completed`.

## Consequences

- **While Postgres is stopped, `terraform plan` on `infra/cloud` fails.** Refresh can't read the database or the Entra admins (`ServerStoppedError: ... is in 'Stopped' state`). Run `start-cloud.sh` before pushing infra changes. CI doesn't start the server itself (yet).
- Azure starts a stopped flexible server on its own after 7 days. The daily stop covers that.
- Storage still bills while stopped. Only compute is saved.
- The frontend is down while AKS is stopped (0011).
- After AKS starts, otel-operator used to crashloop on its own NetworkPolicy blocking the API server. Fixed with `operator.networkpolicy: false` in `infra/k8s/gitops/infrastructure/observability/otel/operator/otel-operator-release.yaml`.
- The schedule is still named `stop-aks-…` although it stops Postgres too. Renaming it would recreate it.
- Verified: `terraform plan` showed the move to 7.6 as in-place runbook updates (no recreate, job links kept). Still to confirm: one manual run of `Stop-PostgreSQL-Server` ends `Completed` with the server `Stopped`.

## References

- https://learn.microsoft.com/en-us/azure/automation/automation-runbook-types
- https://learn.microsoft.com/en-us/azure/automation/runtime-environment-overview
- https://learn.microsoft.com/en-us/azure/automation/whats-new (7.1/7.2 retirement, 7.6 GA)
- https://learn.microsoft.com/en-us/powershell/scripting/install/powershell-support-lifecycle
- https://learn.microsoft.com/en-us/azure/postgresql/configure-maintain/how-to-stop-server
- https://learn.microsoft.com/en-us/azure/postgresql/integration/create-automation-tasks
- https://learn.microsoft.com/en-us/azure/aks/start-stop-cluster
