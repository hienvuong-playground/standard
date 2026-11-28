# 0033 – Cluster alert rules: Microsoft's recommended Kubernetes alerts

## Context

**Disabled since 0035** (alerting moved to Grafana). Kept as reference.

The first alerts to set up (0031). For the cluster layer, Microsoft publishes a recommended set on the Kubernetes metric alerts page. That's Prometheus rules "organized by pod, cluster, and node level", plus 2 platform metric rules. Using it avoids inventing thresholds.

## How it works

Everything is in `infra/cloud/kubernetes-alerts.tf`, and every rule notifies through the Teams action group (0032).

**30 Prometheus rules** in 3 rule groups: Cluster (11), Node (3), Pod (16).

- Source: the ARM template behind https://aka.ms/azureprometheus-recommendedmetricalerts (`Azure/prometheus-collector`, `GeneratedMonitoringArtifacts/Default/recommendedMetricAlerts.json`). The Learn page lists 29. The template also has `KubeNodeDiskUsageHigh`.
- Group names, description, `PT1M` interval and scopes (AMW + cluster) are as in the template.
- The template requires an `actionGroupResourceId` for every rule.
- The file was generated from the template JSON, not typed. Afterwards, all 30 expressions, `for`, severity, `description` annotations and `timeToResolve` values were compared back to the template: no mismatches.
- Fields map 1:1 to `azurerm_monitor_alert_prometheus_rule_group`. `resolveConfiguration` becomes `alert_resolution`, and `actions` becomes `action`.
- Same approach as the recording rules in 0028: unchanged from Microsoft, so the source is easy to compare.

**2 platform metric rules:** node CPU > 95% and node memory working set > 100%, both over 5 minutes, as listed on the same page.

- Microsoft's ARM and Bicep templates don't include them. The page only offers a portal toggle ("Toggle a platform metric rule to enable that rule"), and the page doesn't give the metric names.
- Full definitions come from AMBA, `services/ContainerService/managedClusters/alerts.yaml`:
  - Metrics `node_cpu_usage_percentage` and `node_memory_working_set_percentage`.
  - Average, GreaterThan 95 / 100, window and frequency PT5M, severity 3.
  - Both entries point back to Microsoft's recommended-rules page, and the thresholds and timeframe match it.
- One `azurerm_monitor_metric_alert` with `for_each`.

**Prometheus rules don't fire while the cluster is stopped.** An alert is active only when the expression "results in one or more vector elements". A stopped cluster (nightly, 0019) sends no samples.

## Consequences

- **Rules that never fire here:** `KubeCPUQuotaOvercommit`, `KubeMemoryQuotaOvercommit` and `KubeQuotaAlmostFull`. There's no ResourceQuota in the cluster (`kubectl get resourcequota -A` was empty). They're kept unchanged anyway.
- **Platform rules:**
  - AMBA marks both `verified: false`.
  - They have no split by the `node` dimension, so they look at the cluster-wide value ("Aggregated average CPU utilization measured in percentage across the cluster"), not each node.
- **A node restart floods the channel.** The startup noise 0029 hinted at, now seen.
  - On 2026-11-22 the VMSS was deallocated 16:18–16:21 and started 16:22–16:23 (activity log). The node was `Rebooted`/`NodeReady` at 16:22:47.
  - The cluster has one node, so 74 containers restarted at once (last terminated reason `Unknown` for 70).
  - `KubePodContainerRestart` sums `increase(kube_pod_container_status_restarts_total[1h])` `by (namespace, controller, container, cluster)`, and each resulting series is its own alert. So the 74 restarts made **63 alerts**: 11 controllers have 2 replicas of the same container, which merge into one series. A range query of the expression returned 63 series.
  - Result: 63 Fired cards (16:54) and 63 Resolved cards (17:28, when `increase(...[1h])` dropped to 0). Plus `KubePodFailedState` from the test, 64 alertIds in total.
  - No duplicates: 126 runs read, one card per alertId and state, no retries of the POST.
  - The same will happen after every cluster start (0019). A recurring processing rule doesn't fit, because the start time is manual. Fix: grouped cards (0034).
- **Latency from restart to fired alert: ~32 minutes.**

  | | Step | Took |
  |---|---|---|
  | 16:22:47 | Containers restart | |
  | 16:26 | Expression true (63 series, every minute after) | ~3 min (scrape and ingest) |
  | 16:41 | `for: PT15M` satisfied | 15 min (rule setting) |
  | **16:54:47** | Azure fires the alert (`firedDateTime`) | **~14 min, unexplained** |
  | 16:54:48–51 | Action group → Logic App runs | 1–4 s |
  | 16:54:49–52 | POST to Teams, `202 Accepted` | ~1 s |
  | 16:54 → ~17:07 | Teams posts the cards one by one | 0 to ~13 min, depending on position (0032) |

  - The ~14 min after `for` matched the Job test exactly: due ~16:31, fired 16:44:47. Both fired at second :47.
  - None of the docs read (rule groups, Prometheus alerts, troubleshooting) mention evaluation latency. Not explained yet. A support case would be the next step.
- **Verified end to end (2026-11-22):**
  - `kubectl create job alert-test --image=busybox:1.36 -- sh -c 'exit 1'` in a throwaway namespace `alert-test`. Its failed pods match `KubePodFailedState` (`for: PT5M`, the fastest rule).
  - All the metrics these rules use are in the minimal ingestion profile's kube-state keep list (`kube_pod_status_phase`, `kube_pod_owner`, `kube_job_failed`, `kube_pod_container_status_waiting_reason`, …). `kube_job_status_failed` isn't kept, but no rule uses it.
  - A range query on the AMW showed the expression true every minute from 16:25.
  - Azure fired the alert at **16:44:47**, about 14 minutes after `for` had elapsed. Until then, Alerts API and Resource Graph showed no Prometheus alert at all. The docs read (rule groups, Prometheus alerts, troubleshooting) give no evaluation latency. So don't expect `for` + 1 minute, and don't judge a test before ~20 minutes.
  - The card reached Teams (0032).
  - The namespace was deleted at 16:46. The Resolved card was sent at 17:02:49.
- **Resource Health of rule groups isn't useful here.** All four groups (three alert, one recording) reported "Unavailable… due to an unknown reason" while recording rules were producing data and the alert fired. A Microsoft Q&A thread reports the same mismatch.
- **Not checked:**
  - How metric alerts behave while AKS is stopped (no data).
  - The price of Prometheus rule groups.

## References

- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-metric-alerts
- https://github.com/Azure/prometheus-collector/blob/main/GeneratedMonitoringArtifacts/Default/recommendedMetricAlerts.json
- https://github.com/Azure/prometheus-collector/blob/main/AddonBicepTemplate/recommendedMetricAlerts.bicep
- https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-rule-groups
- https://github.com/Azure/azure-monitor-baseline-alerts/blob/main/services/ContainerService/managedClusters/alerts.yaml
- https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-containerservice-managedclusters-metrics
- https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/prometheus-metrics-scrape-default
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/prometheus-alerts
- https://learn.microsoft.com/en-us/answers/questions/5965141/prometheus-rule-group-resource-health-remains-unkn
- https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/monitor_alert_prometheus_rule_group
