# 0029 – Infrastructure logs in Log Analytics: Container insights "Logs and events"

## Context

AMG had cluster metrics (0028) and application logs (0024), but no infrastructure logs. `OTelLogs` only holds what the backend sends over OTLP. The stdout of every other pod (Flux, cert-manager, the gateway, KEDA, frontend…) and Kubernetes events went nowhere. Troubleshooting meant reading a dashboard, then switching to `kubectl`.

That fails for the cases that matter most:

- The API server keeps events for about one hour (`--event-ttl`, default `1h`). When AKS started at 03:22 (0019), seven containers crashed. By morning, `kubectl get events --field-selector type=Warning` returned nothing.
- `kubectl logs --previous` only keeps the last terminated container.

Microsoft's baseline for AKS turns on four things together: managed Prometheus, Managed Grafana, container logs and control plane logs. With managed Prometheus already on, the recommended log set is the "Logs and events" preset.

## How it works

- **Agent.** The `oms_agent` block on the cluster (`infra/cloud/aks.tf`, with `msi_auth_for_monitoring_enabled = true`) makes AKS deploy `ama-logs` (DaemonSet, reads container stdout/stderr on each node) and `ama-logs-rs` (one replica, reads the Kubernetes API) in `kube-system`. Both are AKS-managed, not Terraform resources.
- **DCR** `MSCI-westeurope-aks-playground` (`infra/cloud/container-insights.tf`). One `extension` data source, `ContainerInsights`, with the three streams of the preset:
  - `Microsoft-ContainerLogV2` → `ContainerLogV2`. One row per log line, with `PodNamespace`, `PodName`, `ContainerName`. `LogMessage` is dynamic, so JSON logs can be queried by field (`LogMessage.level`).
  - `Microsoft-KubeEvents` → `KubeEvents`. Only events that aren't `Normal` (BackOff, FailedScheduling, FailedMount, Unhealthy…), kept 30 days.
  - `Microsoft-KubePodInventory` → `KubePodInventory`. A snapshot of each container every minute (`interval = "1m"`), including restart count and `ContainerLastStatus` (reason, exit code, message).

  Destination: the existing Log Analytics workspace.
- **DCRA** binds the DCR to the cluster. Same pattern as the Prometheus DCRA in 0028.
- **Agent ConfigMap** `container-azm-ms-agentconfig` in `kube-system` (`infra/k8s/gitops/infrastructure/container-insights/`). It enables annotation-based filtering and nothing else. The agent reads the ConfigMap by its fixed name.

## Decision

**Container insights instead of OTel `filelog` + `k8sobjects` receivers.** It's the path Microsoft documents next to managed Prometheus (same addon model as 0028), and AMG's built-in ContainerLogV2 dashboards query these tables directly.

**Port the Terraform template, but not its defaults.** The Learn page's Terraform tab links to `onboarding-msi-terraform` in `microsoft/Docker-Provider`. Taken from it: `oms_agent`, the DCR shape, the DCRA and its name. Changed:

- Streams: only the three above. The template's defaults also turn on `Microsoft-Perf` and `Microsoft-InsightsMetrics`. Those duplicate managed Prometheus, and Microsoft's best practices say not to send Prometheus metrics to Log Analytics. The defaults also include both `ContainerLog` (v1) and `ContainerLogV2`, which would store each line twice, plus inventory tables nobody uses.
- The `-syslog` variant was not used. It adds node OS logs (`Syslog` table), which aren't needed.
- The DCEs and private link resources are left out. They are only created for private link or high log scale mode.

**Exclude the `backend` container.** pino writes each record to stdout, and the OTel auto-instrumentation sends the same record to `OTelLogs`, with `TraceId` and severity. Collecting both would store every backend log line twice. The pod template carries `fluentbit.io/exclude-backend: "true"` (`backend/deploy/deployment.yaml`), and the ConfigMap enables `filter_using_annotations`. Only that container is dropped. `database-setup`, `consumer`, and anything else in the namespace are still collected.

Rejected:

- `transformKql` in the DCR. Microsoft says transformations "should be used only after using other filtering methods", because data leaves the cluster before it's filtered.
- Excluding the `backend` namespace (`exclude_namespaces`). It also drops `database-setup`, `consumer`, and workloads added later. The DCR's own namespace filter doesn't apply to `ContainerLogV2` at all.

**`kube-system` stays excluded.** This is the agent's default. Microsoft excludes system namespaces "to minimize the Log Analytics cost". Their events and pod inventory are still collected. If one system pod's logs are needed later (KEDA, coredns), `collect_system_pod_logs = ["kube-system:<controller>"]` adds just that pod.

Not done yet:

- Control plane logs (diagnostic settings). If added, use `kube-audit-admin`, not `kube-audit` (cost).
- The Basic logs plan for `ContainerLogV2`.
- A dashboard for `KubeEvents` and `KubePodInventory`. AMG has no built-in for them.

**Dashboard "Infrastructure Logs"** (uid `infrastructure-logs`, `infra/grafana/dashboards/infrastructure-logs.json`). A copy of the built-in "Azure / Insights / Containers / ContainerLogV2 Analytics Tier", because the table is on the Analytics plan. Built-ins can't be changed in place (0023). The `infrastructure-*.json` resource in `infra/grafana/dashboards.tf` picks it up, and the `infrastructure` tag puts it on Home (0025).

Queries are unchanged except for the built-in bugs already found in 0024:

- Log table: `$__timeFilter`, `LogLevel in~ ($sev)` instead of the `isempty($sev)` check, and `top 100 by TimeGenerated desc` instead of `take 100`.
- "Log Volume by Computer" and "by Namespace": `$__timeFilter` added. They summed the whole retention.
- Volume panels: unit `decbytes`, since `_BilledSize` is bytes.
- Level links: `sub`, `rg`, `res`, `ns` instead of variable names from another dashboard, and TOTAL sets `var-sev=$__all`.
- The 18 Explore links were removed.

## Consequences

- **Expected volume.** Container stdout measured over one hour (via `kubectl logs --since=1h`) was about 6.6 MB/h raw in total:

  | Source | Approx. MB/h | Collected? |
  |---|---|---|
  | `kube-system` | 1.7 | no (default exclusion) |
  | `backend` | 0.33 | no (annotation) |
  | `fluxconfig-agent` / `fluxconfig-controller` and their `fluent-bit` sidecars (AKS GitOps extension) | 3.4 | yes |

  The `fluxconfig-*` containers logged about 40k lines in 24 h, all `"LogLevel":"Information"`. They are more than half of `ContainerLogV2` and are the first thing to filter if cost matters. The Flux controllers themselves (`kustomize-controller` etc.) are about 0.6 MB/h and are the useful part.
- **Cost.** In West Europe, Analytics logs cost $2.99/GB after the first 5 GB per month per billing account (Azure Retail Prices API). Before this change the workspace took in about 49 MB/day. Expect a few GB per month more. `KubePodInventory` volume hasn't been measured.
- **A backend crash may leave no message.** On `uncaughtException`, `backend/src/index.js` logs `fatal` and calls `process.exit(1)` right away. The OTel batch probably doesn't flush, and stdout is now excluded. This is not verified. `KubePodInventory` still shows the exit code. Errors Node prints itself before the handlers are registered go to stderr. The annotation drops stderr too.
- **Existing noise, not addressed here.** 88% of `OTelLogs` rows in 24 h were kube-probe requests (`/readyz`, `/livez`), and the backend runs with `LOG_LEVEL=trace`. The frontend's nginx access log is mostly probes as well. The backend part is handled in [0030](0030-health-check-noise.md).
- **The `fluent-bit` sidecars of the GitOps extension log** `ManagedIdentityCredential authentication failed … identity isn't assigned` repeatedly. That is the extension sending its own telemetry to Microsoft. Flux reconciles fine. Not investigated.
- **Verified:**
  - A local `terraform plan` showed the cluster updated in place (`oms_agent` added) and the DCR and DCRA created.
  - The CI apply succeeded. `ama-logs` and `ama-logs-rs` are running, the ConfigMap is applied, and the new backend pod has the annotation.
  - All 18 queries of Infrastructure Logs run against the workspace with real variable values (1 h). Log Volume (7,005,880 bytes) equals the sum by node. The table is newest first.
  - Data arrives in all three tables. In the 24 h to 2026-11-21 17:00 UTC there were 124,244 `ContainerLogV2` rows, 34,542 `KubePodInventory` rows (from 11:28 UTC) and 71 `KubeEvents` rows (from 11:22 UTC).

## References

- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable
- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/best-practices-containers
- https://learn.microsoft.com/en-us/azure/aks/monitor-aks
- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-data-collection-configmap
- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-transformations
- https://github.com/microsoft/Docker-Provider/tree/ci_prod/scripts/onboarding/aks/onboarding-msi-terraform
- https://github.com/microsoft/Docker-Provider/blob/ci_prod/kubernetes/container-azm-ms-agentconfig.yaml
- https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/
- https://azure.microsoft.com/en-us/pricing/details/monitor/
