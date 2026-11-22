# 0033 – Cluster alert rules: Microsoft's recommended Kubernetes alerts

## Context

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
- **Startup noise is possible.** 0029 saw seven containers crash when AKS started. Restart and crash-loop rules can catch that every morning. If it does, an alert processing rule with a recurring window is the fix (0031).
- **Not checked:**
  - How metric alerts behave while AKS is stopped (no data).
  - The price of Prometheus rule groups.
  - A real fired alert end to end. CI applied the rules (0032, run 45003455703), but nothing has fired yet.

## References

- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-metric-alerts
- https://github.com/Azure/prometheus-collector/blob/main/GeneratedMonitoringArtifacts/Default/recommendedMetricAlerts.json
- https://github.com/Azure/prometheus-collector/blob/main/AddonBicepTemplate/recommendedMetricAlerts.bicep
- https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-rule-groups
- https://github.com/Azure/azure-monitor-baseline-alerts/blob/main/services/ContainerService/managedClusters/alerts.yaml
- https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-containerservice-managedclusters-metrics
- https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/monitor_alert_prometheus_rule_group
