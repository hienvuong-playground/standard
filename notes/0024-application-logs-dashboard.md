# 0024 – Application Logs dashboard, from the built-in ContainerLogV2 Analytics Tier

## Context

Logs reach Log Analytics over OTLP (DCR stream `Microsoft-OTel-Logs`, `infra/cloud/monitor.tf`) and land in the `OTelLogs` table. There is no Container Insights, so `ContainerLogV2` is empty. The new dashboard keeps the layout of AMG's built-in "Azure / Insights / Containers / ContainerLogV2 Analytics Tier" and rewires its queries:

- Title "Application Logs", uid `application-logs`, folder Observability (0025).
- First made through the Grafana HTTP API, now in Terraform: `infra/grafana/dashboards/application-logs.json` (0026).

## Decision

**Mapping.** Every query starts with `OTelLogs` plus an `extend` that rebuilds the ContainerLogV2 columns, so the rest of each query stays as it was:

- `PodNamespace`, `PodName`, `ContainerName`, `Computer` come from `ResourceAttributes["k8s.namespace.name" / "k8s.pod.name" / "k8s.container.name" / "k8s.node.name"]`. The table's own `K8sNamespaceName`, `K8sPodName` and `K8sContainerName` columns are empty.
- `LogMessage` = `Body`.
- `LogLevel` comes from OTel `SeverityNumber` ranges: 1–4 trace, 5–8 debug, 9–12 info, 13–16 warning, 17–20 error, 21–24 critical, anything else unknown. These names match the level filters on the stat panels.
- Dropped the AKS-cluster `_ResourceId` filter. In `OTelLogs`, `_ResourceId` is the App Insights resource.
- Dropped the cluster variable. The workspace variable now lists the Log Analytics workspaces in the selected resource group, instead of reading the AKS omsagent addon.

**Built-in bugs, fixed.** The grafana.com copy (#24093, revision 1) has the same panels and the same bugs.

- **Log table:**
  - `isempty($sev)` breaks on the default value All, because a multi-value variable expands to a list. Replaced with `| where LogLevel in~ ($sev)`.
  - It had no time filter, and `take 100` returns arbitrary rows. Now `$__timeFilter` plus `top 100 by TimeGenerated desc`.
- **"Log Volume by Computer" and "by Namespace"** had no `$__timeFilter`, so they summed the whole retention (about 115 MB, against about 3.3 MB in an hour). Added it.
- **Volume panels:** `_BilledSize` is in bytes, but the unit was `short` (shown as "3.28 Mil") and the column was named `…MB`. Unit is now `decbytes`.
- **Level stat links:**
  - TOTAL set `var-sev=` (empty), which breaks the table query. Now `$__all`.
  - The links used variable names from another dashboard (`Subscription`, `Cluster`, `Namespace`). Now `sub`, `rg`, `ns`.
  - Removed 18 "Explore" links that queried ContainerLogV2 with `KubernetesMetadata`.

## Consequences

- **Only OTel-instrumented apps appear.** Today that is just the backend. Seeing the stdout of other pods would need a `filelog` receiver in the collector.
- **Verified:**
  - All 18 queries and both variables run without errors.
  - The volume panels agree with each other.
  - The table is sorted newest first.
  - The `sev=error` filter returns only error rows.
- **Not fixed:**
  - "Average Log Rate/Min" ignores minutes with no logs.
  - The table shows only `Body`, with no trace id.

## References

- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-logs-schema
- https://opentelemetry.io/docs/specs/otel/logs/data-model/#field-severitynumber
- https://grafana.com/grafana/dashboards/24093
