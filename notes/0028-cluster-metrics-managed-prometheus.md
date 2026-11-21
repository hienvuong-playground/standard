# 0028 – Cluster metrics in AMG: managed Prometheus and infrastructure dashboards

## Context

The built-in "Kubernetes / Compute Resources / …" dashboards on AMG (0021) were empty. They query cluster metrics (`container_*{job="cadvisor"}`, `node_*{job="node"}`, `kube_*`) with a `cluster` label. They also query precomputed series such as `cluster:node_cpu:ratio_rate5m`. The `cluster` dropdown is `label_values(up{job="cadvisor"}, cluster)`, so without that series every panel is empty.

Nothing sent cluster metrics to the Azure Monitor workspace (AMW). It held only the backend's OTel metrics (22 metric names, 0022). On AKS, `azureMonitorProfile.metrics.enabled` was `false`. AMG doesn't read the self-hosted Prometheus.

## How it works

- **Agent.** `monitor_metrics {}` on the cluster (`infra/cloud/aks.tf`) makes AKS deploy the `ama-metrics*` pods in `kube-system`. By default they scrape cadvisor, node-exporter, kubelet, kube-state-metrics and network observability. On Cilium the network jobs are `networkobservability-cilium` and `networkobservability-hubble`. The pods are AKS-managed, not Terraform resources.
- **DCR** (data collection rule, `infra/cloud/managed-prometheus.tf`). It says what is collected and where it goes:
  - Source `prometheus_forwarder`: the stream `Microsoft-PrometheusMetrics` that the agent emits. The name is fixed by Microsoft.
  - Destination: the AMW.
  - `data_flow`: connects the stream to the destination.

  No code references the agent. Agent and DCR match by stream name.
- **DCRA** (DCR association). Binds the DCR to the cluster's resource ID. Without it the agent runs but has nowhere to send.
- **DCE** (data collection endpoint). The agent's configuration and ingestion endpoint. Prometheus requires one. How the agent finds its DCR is not described step by step in the docs. Most likely it asks the DCE's configuration endpoint for the DCRs associated with its cluster.
- **Recording rules.** Rule groups that AMW evaluates every minute. Each rule stores the result of a query as a new series. The dashboards query those series directly. For example, `cluster:node_cpu:ratio_rate5m` is computed from `node_cpu_seconds_total`.

## Decision

**Port Microsoft's `AddonTerraformTemplate` instead of writing our own.** The portal, CLI and ARM onboarding create these resources for you. Terraform doesn't. The Learn page's Terraform tab sends you to this template: "Download all files under AddonTerraformTemplate." It also requires that "The Azure Monitor workspace and Azure Managed Grafana workspace must already be created", which is the case here (0021). (https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable?tabs=terraform#enable-prometheus-metrics-on-an-aks-cluster)

Copied:

- DCE, DCR and DCRA, named `MSProm-<location>-<cluster>` like Microsoft's.
- `monitor_metrics {}`. The template passes two optional allowlists that default to `null`.
- Rule groups Node (11 rules), Kubernetes (19) and UX (18). Expressions are unchanged, checked by diff.

Left out:

- Resource group, AKS, AMW, AMG, and `Monitoring Data Reader` on the AMW. They already exist, and the role is subscription-wide (0021).
- The "mismatch" DCE and DCRA. They are only for a private cluster in a different region than the AMW.
- The three Windows rule groups. The template disables them by default, and there are no Windows nodes.
- Control plane metrics (apiserver, etcd). The template defaults them to off, and no built-in dashboard uses them. They would need `azapi_update_resource`.

Deviations: references point at our resources, the DCE/DCR/DCRA Terraform names are prefixed `managed_prometheus_`, and the redundant `depends_on` is dropped.

The UX rules (`ux:*`) are not used by any AMG dashboard. All 21 Kubernetes dashboards were checked. They are kept because the template enables them. They probably feed the AKS portal views (not verified).

**Register `Microsoft.AlertsManagement` in `infra/init/bootstrap.sh`.** Rule groups live in that namespace. The first apply failed with `MissingSubscriptionRegistration`, and azurerm 5.x registers nothing. `Microsoft.Dashboard` (AMG) was added at the same time. It was registered on the subscription but missing from the list, so a rebuild would have failed. On the current subscription it was registered once by hand.

**Copy the main dashboards into Terraform** (`infra/grafana/dashboards/infrastructure-*.json`, one `for_each` resource in `infra/grafana/dashboards.tf`). Built-ins are provisioned by Azure and can't be changed in place (0023).

- The seven copies: Cluster, Namespace (Pods), Namespace (Workloads), Workload, Pod, Node (Pods), and Networking (from "Networking / Clusters"). Titles follow "Infrastructure …" (0025).
- Tags `playground` and `infrastructure`, so the Home page's Infrastructure list picks them up.
- Drill-down links point to the copies. A "Playground dashboards" top-bar link was added.
- Panels and queries are unchanged, including those that are empty on Cilium, so they can be edited later.

Rejected:

- Tagging the built-ins: they are provisioned, and the tags wouldn't be in Terraform.
- A dashboard list by tag `kubernetes-mixin`: it shows the Windows dashboards, misses Networking, and can't be edited.

## Consequences

- **The agent needed about 7 minutes after the DCRA to get its config.** Until then it logs "No configuration present for the AKS resource" and drops data. Wait instead of restarting pods.
- **Panels based on recording rules need about 5 more minutes,** because the rules use `[5m]` windows.
- **A commit that only touches `init/` skips `terraform-cloud` in CI** (paths filter). The failed run had to be re-run by hand, which needs repo admin.
- **Expected empty panels:**
  - Limits, memory requests and CPU throttling: the backend only sets `requests.cpu`.
  - Kubelet error panels: there are no errors.
  - Networking drops by reason or node: the query is `rate(...) > 0`, so it only shows while drops happen.
  - Networking TCP, UDP and interface panels: they use Retina-only metrics. At node level Cilium only exposes `cilium_forward_*` and `cilium_drop_*`.
  - DNS dashboards: on Cilium they need a Cilium FQDN network policy.
  - Pod Flows: managed Prometheus excludes `hubble_flows_processed_total` by default.
- **Drops seen while checking:** about 55 ICMPv6 packets per 15 min with reason `unsupported l3 protocol`. The cluster is IPv4-only, so this is harmless noise. There were also a few earlier ICMPv4 `policy_denied` drops to `flux-system` and `observability` pods.
- **The copies don't follow Microsoft's updates** to the built-ins.
- **AMW bills per ingested sample.** This is a new cost, not measured yet.
- **Verified:**
  - After the apply, AMW has the jobs `kubelet`, `node` and `cadvisor`. `kube_pod_info` covers 65 pods. `cluster:node_cpu:ratio_rate5m` ≈ 0.062.
  - Every panel query of the seven built-ins was run against AMW with real variable values. All 22 panels of Cluster return data, and every empty panel is in the list above.
  - A local plan of the `grafana` stack showed 7 to add and 0 to change.

## References

- https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-metrics-overview
- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable
- https://github.com/Azure/prometheus-collector/tree/main/AddonTerraformTemplate
- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/prometheus-metrics-scrape-default
- https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-rule-groups
- https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-rule-overview
- https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-endpoint-overview
- https://learn.microsoft.com/en-us/azure/aks/container-network-observability-metrics
