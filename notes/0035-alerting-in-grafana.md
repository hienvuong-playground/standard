# 0035 – Alerts move to Grafana (AMG), Azure Monitor alerts disabled

## Context

0031 put alerting in Azure Monitor. After running it (0032–0034), Grafana came out clearly ahead:

- **The UI is nicer and easier to use** than Azure's alert pages.
- **Alert state is visible.** Grafana shows each rule and instance as Normal, Pending, Alerting, Recovering, No Data or Error. Azure showed nothing until an alert fired: during the `for` period, and during the unexplained ~14 minutes after it, "Alerts API and Resource Graph showed no Prometheus alert at all" (0033).
  - Example from the first test (2026-11-28): one failing Job made two rules true at 08:04:30. `KubePodFailedState` (`for` 5m) fired at 08:09:30. `KubeJobStale` (`for` 6h, "Job stuck") stayed Pending. In Grafana you see which rule is waiting and since when. In Azure both would have been invisible.
- **Faster, and the delay is explainable.** Same test as 0033 (a Job that exits 1, `KubePodFailedState`), counted from the moment the expression turned true:
  - Azure: true at 16:25, fired 16:44:47. About 20 minutes, ~14 of them unexplained.
  - Grafana: true (Pending) 08:04:30, firing 08:09:30, sent to Teams 08:10:00. About 5.5 minutes: the 5-minute `for` plus the 30-second `group_wait`.
  - Every part of Grafana's delay has a name and a setting:

    | Part | How long | Where it's set |
    |---|---|---|
    | Scrape + ingest into the AMW | not measured alone; ~3 min from restart to true expression in 0033 | — |
    | Query offset | 1 min (value at now-1m) | import default `rule_query_offset` |
    | Evaluation interval | up to 60 s | `interval_seconds = 60` |
    | Pending (`for`) | per rule, 5 min to 6 h | Microsoft's rules |
    | `group_wait` | 30 s | Grafana default |
    | Teams posting | async, 25 cards per 300 s (0032) | Teams |

  - Expected after a short node outage like 0033's (VMSS off ~4 minutes, 63 restart alerts): ~3 + 1 + 15 (`KubePodContainerRestart`'s `for`) + <1.5 ≈ 20 minutes, as one grouped card. Azure took ~32 minutes, and Teams then needed ~13 more minutes to post 63 single cards. Not measured yet with Grafana.
  - **The nightly stop/start (0019) doesn't trigger that.** The rule uses `increase(...[1h])`. After a night off, the 1-hour window only holds samples from after the start. Checked on 2026-11-28 (start 05:50): the summed restart counter was 5 before the stop and 2 after the start, flat until 08:19. The rule's expression had 0 series > 0 since 05:50. So a start that comes up healthy gives no alerts. This is about rules that count changes over a window (`increase`). Rules that look at the current state (`KubePodCrashLooping`, `KubePodNotReadyByController`, `KubeDeploymentReplicasMismatch`, `KubePodFailedState`…) still fire after any start if something stays broken, after their `for`.
- **Grouping is built in.** Azure action groups don't group. 0034 had to rebuild Alertmanager's grouping with Logic Apps Batch, still without per-group state, repeat or inhibition.
- **A default message template.** Grafana sends `default.title` / `default.message` to Teams as is. Azure sends the common alert schema, so the card had to be built by hand in a Logic App (0032), and even listing labels needed an `xml()`/`xpath()` workaround because workflow expressions have no `keys()` (0034). That code is hard to read and to change.

The extra Azure Monitor SLA on the alert path (0031) was judged not worth this. AMG has a 99.9% SLA of its own, though it only covers "HTTP requests to perform operations", not alert evaluation (0031).

## Decision

**Rules, contact point and notification policy live in AMG, managed by Terraform in `infra/grafana/alerts.tf`.** The Azure Monitor alerts are disabled, not deleted, to keep them as reference and to make switching back cheap:

- `rule_group_enabled = false` on the three Prometheus rule groups and `enabled = false` on the two metric alerts (`infra/cloud/kubernetes-alerts.tf`). "Disabled rule groups will still be created, but the rules will only be run if the group is enabled."
- `enabled = false` on both Logic Apps (`infra/cloud/teams-alerts.tf`). The batch workflow's trigger polls every minute, and Consumption bills a polling trigger "whether or not the trigger fires", while "Disabled logic apps or workflows" aren't billed.
- Action group, Key Vault and secret stay. Recording rules (`managed-prometheus.tf`) aren't alerts and stay.

Rejected: keeping Azure Monitor (reasons above).

## Two layers: evaluating rules vs. notifying

This was the confusing part, so spelled out:

- **Evaluation** runs the query on a schedule and decides which alerts are firing.
- **Notification** groups, waits, repeats and sends. That's Alertmanager's job. Grouping belongs here, not to whoever evaluates.

Azure's evaluation was fine (rules next to the data). Azure's gap was the notification layer: action groups don't group.

Grafana names two ways to place evaluation:

- **Data source-managed:** rules are "stored in the data source, such as in Prometheus or Loki, rather than in Grafana". The backend's ruler evaluates and sends to its own Alertmanager (Mimir's "adds multi-tenancy support and horizontal scalability to the Prometheus Alertmanager", and Grafana's UI can manage its policies). So grouping exists here too. Grafana can only create these rules for "Mimir and Loki data sources".
- **Grafana-managed:** rules are stored in Grafana, Grafana queries the data source over its API, and its built-in Alertmanager ("extends the Prometheus Alertmanager") notifies.

The AMW has no Ruler API (Microsoft only documents rule groups as Azure resources), so only Grafana-managed is possible here. The Azure rule groups were the data source-managed equivalent.

## How the rules are built

**30 Prometheus rules (Cluster / Node / Pod groups):** Microsoft's recommended set (0033), in exactly the shape Grafana's own Prometheus import produces (`pkg/services/ngalert/prom/convert.go`):

| Node | Content |
|---|---|
| `query` | PromQL, instant, time range 660s→60s (1m offset, the import default) |
| `prometheus_math` | `is_number($query) \|\| is_nan($query) \|\| is_inf($query)`, which turns every returned series into 1 |
| `threshold` (condition) | `> 0` |

- The math node is needed because the PromQL already holds the threshold. A query/threshold pair alone would miss a series whose value is 0, e.g. `KubePersistentVolumeInodesFillingUp` when inodes are fully used. Grafana: "to preserve Prometheus no data behavior".
- `for` and `keep_firing_for` are copied (Azure's `timeToResolve`: "resolve alerts if the rule condition is no longer true during the Time to auto-resolve period"). `missing_series_evals_to_resolve = 1`, as in the import.
- `$value` in the annotations still means the query value: "If there's exactly one datasource node, use its numeric value… compatible with Prometheus templating" (`pkg/services/ngalert/state/template/template.go`, 13.2.4).
- **Time range 660s→60s.** The values count back from the evaluation time. An instant query "evaluates the expression once, at the end of the selected time range", so only `to` matters: the rule asks for the value at now-1m ("an offset of `1m` adjusts the query's time range to `To: now-1m`"). `from` has no effect. 660 = the import's 600s default + offset, kept so the rule matches the import. Ranges inside the PromQL (`[1h]`) count back from that point.
- **If the newest samples aren't ingested yet:** an instant query takes "the newest sample that is less than the lookback period ago" (5 minutes), so it uses a slightly older sample, not nothing. Only after more than 5 minutes without data do the series disappear. Then rules are Normal, and firing alerts resolve (`missing_series_evals_to_resolve = 1`), as in Prometheus. That's a broken metrics pipeline, which these rules don't detect (the AMW still answers queries, so there's no error either). The real AMW ingestion delay hasn't been measured.

**No Data = OK.** For these rules an empty result means healthy: the query only returns series that are in trouble. Grafana's default No Data handling assumes the opposite (the query returns numbers when healthy, so empty = broken) and would create a `DatasourceNoData` alert. Here "healthy" and "no data at all" look the same, so No Data can't tell them apart.

- Rewriting the rules Grafana-style (threshold as its own node, No Data default) was rejected. 11 of 30 rules use the comparison as a filter inside `and`/`unless`/`or`/`changes()`/`!=`, so the logic would have to be rewritten. Several metrics only exist with their objects (no ResourceQuota in the cluster, so the quota rules would be NoData forever). The nightly stop (0019) would raise NoData daily. The missing-data guide itself suggests Normal for "intermittent data".
- Grafana-style rules with the default No Data are the right choice for rules written here from scratch (e.g. application golden signals), where the query always returns data when healthy.
- So these rules don't detect "the cluster is gone". That needs its own rule, not added yet.

**Query errors: Error (Grafana's default), not OK.** A failing query makes a `DatasourceError` alert with `rulename`, `datasource_uid`, `ref_id` and the error text in the `Error` annotation.

- First version followed Grafana's connectivity-errors guide: OK on the rules plus one dedicated rule (`AzureMonitorWorkspaceUnreachable`, `vector(1)`, errors → Alerting). The guide says to "decide it per rule, based on whether that rule already has a dedicated connectivity alert covering it". Its premise is that query errors come from the data source (timeouts, unreachable). It doesn't cover a rule whose own query is wrong.
- `KubePVUsageHigh` showed exactly that case (below): the AMW was fine, the health rule was green, and the rule failed on every evaluation without anyone being told.
- With Error on every rule, the dedicated rule adds nothing and was removed. If the AMW is down, every rule errors. Several rules have `for` 5m ("The Error state honors the pending period"), so `DatasourceError` arrives as fast as the health rule did. `DatasourceError` has no `cluster` label, so with `group_by = [alertname, cluster]` all of them come as one card.
- Short glitches stay quiet: Error needs "repeated failures (default: `3`)" and then the rule's `for`.
- Grafana's meta-monitoring page suggests alerting on `grafana_alerting_alerts{state="error"} > 0`. That needs Grafana's own metrics, which AMG probably doesn't expose (not checked).
- Tested 2026-11-28 with a throwaway rule using `KubePVUsageHigh`'s old query, `for` 0, interval 10s: created 08:44:21, Error and `DatasourceError` at 08:44:30, sent to Teams 08:45:02. The rule's `health` shows `error` too (with OK it showed `ok`). Rule and folder deleted.

**2 platform metric rules (Platform group):** node CPU > 95 % and memory working set > 100 %. They're part of Microsoft's recommended set (0033), but they're Azure platform metrics of the AKS resource, not Prometheus data, so they go through the Azure Monitor data source. Query with `PT1M` grain over the last 5 minutes (here `from = 300` does matter) → Reduce mean (drop non-numeric) → threshold. That matches AMBA's "Average over PT5M", every 5 minutes. No Data = OK (AKS stopped). Errors = Error: with two rules a `DatasourceError` per rule is fine. Subscription, resource group and cluster name come from `cloud` outputs (`subscription_id`, `resource_group_name`, `aks_cluster_name`).

Both data sources were created by Azure and use managed identity (`msi`), which alerting needs: rules "run as background processes without a user context".

## Notifications

- Contact point `teams` → the same Teams Workflows webhook (0032). The Grafana docs use the template "Post to a channel when a webhook request is received". The current workflow is "Send webhook alerts to a channel". If the test card doesn't show, create a new workflow and follow 0032's path (tfvars → init → GitHub secret).
- Default policy: `group_by = ["alertname", "cluster"]` (Alertmanager's example, as in 0034). It groups by **labels** ("same exact label values for the labels configured in the `Group by` option"). Annotations are only text. Nothing here sets the two labels:
  - `alertname` is reserved: "automatically added by Grafana: `alertname`: the name of the alert rule".
  - `cluster` comes from the data. Azure appends "the last part of the cluster's resource ID… to every time series", and Grafana copies query labels onto the alert. Checked: every `up` series in the AMW has `cluster="aks-playground"`.
  - The platform rules (Azure Monitor) and `DatasourceError`/`DatasourceNoData` have no `cluster` label. They group by `alertname` only. Timers are Grafana's defaults (30s / 5m / 4h). This replaces the Logic App batch and adds what it lacked: per-group state, `group_wait` from the first alert, repeat.
- **The webhook URL is in `grafana.tfstate`.** The contact point's `url` is `Sensitive`, not write-only, so an ephemeral variable can't be used. Accepted. CI passes `TEAMS_ALERTS_WEBHOOK_URL` to the `terraform-grafana` job (`infra/.github/workflows/deploy.yml`).

## KubePVUsageHigh: Microsoft's expression is wrong

```
avg by (namespace, controller, container, cluster)(((kubelet_volume_stats_used_bytes{job="kubelet"} / on(namespace,cluster,pod,container) group_left kubelet_volume_stats_capacity_bytes{job="kubelet"}) * on(namespace, pod, cluster) group_left(controller) label_replace(kube_pod_owner, "controller", "$1", "owner_name", "(.*)"))) > .8
```

- It joins on `pod` (and `container`), but `kubelet_volume_stats_*` only have `namespace` and `persistentvolumeclaim` (Kubernetes metrics reference). Microsoft's own default `kubelet` scrape config only adds `metrics_path`, `instance` and `kubernetes_io_os` (`otelcollector/shared/configmap/mp/testdata/default-linux-ds.yaml`). The AMW series have no `pod` label (checked).
- Result, checked on the AMW:
  - A namespace with one PVC: the first join works (0.0002). The full expression **without** `> .8` is empty, because the second join compares `pod=""` with `kube_pod_owner`'s real pod names.
  - A namespace with 2+ PVCs (`observability` has three): "many-to-many matching not allowed".
  - So it's empty or an error, never a series. Even a full PV wouldn't alert. The description uses `{{ $labels.pod }}`, which never exists.
- Same expression everywhere Microsoft publishes it: the deployed Azure rule group (`az resource show … KubernetesAlert-RecommendedMetricAlertsaks-playground-Pod-level`), `GeneratedMonitoringArtifacts/Default/recommendedMetricAlerts.json` (target of `aka.ms/azureprometheus-recommendedmetricalerts`), `AddonBicepTemplate/recommendedMetricAlerts.bicep:603`. The Azure portal's rule editor shows the same "many-to-many matching" error on the Azure rule.
- Why nobody notices: in Azure a failing rule is silent (at most Resource Health "Unavailable… unknown reason", as seen in 0033). Microsoft's e2e test has the `kubelet_volume_stats_*` metrics commented out (`otelcollector/test/ginkgo-e2e/querymetrics/query_metrics_test.go`). The upstream mixin in the same repo joins on `(namespace, persistentvolumeclaim)`.
- **Fix here:** `kubelet_volume_stats_used_bytes{job="kubelet"} / kubelet_volume_stats_capacity_bytes{job="kubelet"} > .8`, per PVC, with the description using `persistentvolumeclaim` and `namespace`. Both metrics have the same labels, so they divide one to one. Without the threshold it returned the three PVCs (1.1 %, 0.02 %, 0.02 %). `for` 15m and the 80 % threshold are Microsoft's.
- Not checked: whether the portal's "enable recommended alerts" toggle uses this same template.
- Reported upstream: https://github.com/Azure/prometheus-collector/issues/1828 (2026-11-28 09:36). The proposal there also adds `unless on(namespace, persistentvolumeclaim) kube_persistentvolumeclaim_access_mode{access_mode="ReadOnlyMany"} == 1`, like `KubePersistentVolumeFillingUp`. The repo's stale bot marks issues after 7 days and closes them 5 days later, and in 2026 no outside issue got a maintainer reply, so comment within 7 days to keep it open.
- Deployed 2026-11-28 (infra `6c0bd33`): 32 rules, all `exec_err_state = Error`, all `health=ok`. `KubePVUsageHigh` is `Normal (NoData)` (no PVC above 80 %), no error. Microsoft fixed a similar wrong-label bug after an issue (#1125, `KubePodReadyStateLow`), so this one is worth reporting too.
- The Azure Monitor query needs `query_type = "Azure Monitor"` on its `data` block. Grafana sets it from the model, and without it every plan showed `query_type = "Azure Monitor" -> null` (fixed in `13cf1d6`, next plan: "No changes").

## Consequences

- **Microsoft doesn't support it:** "Using Grafana alerts with an Azure Monitor service isn't officially supported by Microsoft." That's a support statement, not a technical limit.
- **Alert rules "share the same compute resources and query-throttling limits with dashboard rendering"** in AMG. The X1 limit is 500 rules. We use 32.
- **If AMG itself is down, nothing alerts and nobody is told.** Not covered yet.
- **Rules provisioned by Terraform can't be edited in the UI.** Same as dashboards (0026).
- **Every rule in AMG goes to Teams through the default policy,** including rules made by hand in the UI.
- **Teams throttling stays** (Flow bot, 25 per 300 s, 0032), but grouping keeps the card count low.
- **Data source uids are hard-coded** (`amw-playground`, `azure-monitor-oob`). Azure creates both data sources: the AMW integration (`azure_monitor_workspace_integrations`) makes `Managed_Prometheus_amw-playground` with uid = the AMW name, and AMG makes `Azure Monitor` ("oob") at creation. No doc found for that naming. If a uid changes, the `grafana_data_source` lookup fails at plan, so nothing points at the wrong source silently. Open options:
  - Manage our own data sources in Terraform (url from `amw_query_endpoint`, MSI). The rules then reference them, but AMG gets a duplicate of each unless the integration is removed, which may take other Azure-made things with it (to check).
  - Derive the uid from the AMW name. Rejected: it still relies on the undocumented naming.
- **Verified before deploy (2026-11-28):**
  - The template on `main` still matches all 30 expressions in `kubernetes-alerts.tf`.
  - The template was POSTed to Grafana's import API (`/api/convert/prometheus/config/v1/rules/convert-test`, rules paused) on AMG. The output was compared with the Terraform rules: 30/30 match on group, `for`, `keep_firing_for`, labels (minus `__converted_prometheus_rule__`), annotations, the query model, time range, math, threshold, condition, no-data/error state and missing-series setting. The only differences were `60m` vs `1h`, which the provider treats as equal. The test folder was deleted. Changed on purpose afterwards: `exec_err_state` (Error, not OK) and `KubePVUsageHigh`'s expression.
  - `terraform validate` for both stacks.
- **Verified after deploy (2026-11-28, CI run 46090482196):**
  - `cloud`: "0 added, 7 changed". The three alert rule groups report `enabled=false` (the recording rule groups stay on), both metric alerts `False`, both Logic Apps `Disabled`. `grafana`: "7 added".
  - All 33 rules (then incl. the health rule) `health=ok`, `inactive`, no `DatasourceNoData`. But with `exec_err_state = OK` a failing query still shows `ok`, and only the instance says `Normal (Error)` (see `KubePVUsageHigh` below). Evaluating the rules through `/api/v1/eval` returned real values: node CPU 7 %, memory 72.8 %, `vector(1)` = 1. `KubePodContainerRestart` returned nothing (healthy).
  - **Microsoft's `KubePVUsageHigh` can never fire, on any cluster with Azure Managed Prometheus.** Fixed here, see "KubePVUsageHigh" below.
  - **Nightly stop/start gives no alerts.** All 30 expressions replayed over 05:50–08:00 on 2026-11-28 (start at 05:50): 28 never true, `KubeJobStale` true for 1 minute in `kube-system` (its `for` is 6 h), `KubePVUsageHigh` errors.
  - Contact point test (`/apis/notifications.alerting.grafana.app/v1beta1/namespaces/default/receivers/<uid>/test`; the old `/api/alertmanager/grafana/config/api/v1/receivers/test` returns 410) at 08:02:25: `success`.
  - Real alert: `kubectl create job alert-test --image=busybox:1.36 -- sh -c 'exit 1'` in a throwaway namespace. Timeline above. The instance had `alertname`, `cluster`, `controller`, `namespace`, `severity` and `grafana_folder`. Teams `lastNotifyAttempt` 08:10:00, no error. The namespace was deleted at 08:11:11. The rule went Recovering (query empty) at ~08:13:50, so `keep_firing_for` applies before the missing-series eviction. Normal at ~08:29, after the 15-minute `keep_firing_for`. Resolved notification at 08:30:00.
  - AMG runs two servers, and each keeps its own notification status. `lastNotifyAttempt` on `/api/alertmanager/grafana/config/api/v1/receivers` therefore shows the real time or `0001-01-01` depending on which server answers.
  - kubectl: `kubelogin --login azurecli` picks the az default tenant (WORK) and gets `Unauthorized`. Workaround: `az account get-access-token --subscription Playground --resource 6dae42f8-4368-4678-94ff-3960e28e3630` and `kubectl --token`.

## References

- https://grafana.com/docs/grafana/latest/alerting/fundamentals/alert-rules/
- https://grafana.com/docs/grafana/latest/alerting/alerting-rules/create-data-source-managed-rule/
- https://grafana.com/docs/grafana/latest/alerting/fundamentals/notifications/alertmanager/
- https://grafana.com/docs/grafana/latest/alerting/set-up/configure-alertmanager/
- https://grafana.com/docs/mimir/latest/references/architecture/components/alertmanager/
- https://grafana.com/docs/mimir/latest/references/architecture/components/ruler/
- https://grafana.com/docs/grafana/latest/alerting/alerting-rules/alerting-migration/
- https://github.com/grafana/grafana/blob/v13.3.2/pkg/services/ngalert/prom/convert.go
- https://github.com/grafana/grafana/blob/v13.2.4/pkg/services/ngalert/state/template/template.go
- https://grafana.com/docs/grafana/latest/panels-visualizations/query-transform-data/expression-queries/
- https://grafana.com/docs/grafana/latest/alerting/fundamentals/alert-rules/queries-conditions/
- https://grafana.com/docs/grafana/latest/alerting/fundamentals/alert-rule-evaluation/
- https://grafana.com/docs/grafana/latest/alerting/fundamentals/alert-rule-evaluation/nodata-and-error-states/
- https://grafana.com/docs/grafana/latest/alerting/guides/missing-data/
- https://grafana.com/docs/grafana/latest/alerting/best-practices/connectivity-errors/
- https://grafana.com/docs/grafana/latest/alerting/fundamentals/notifications/group-alert-notifications/
- https://grafana.com/docs/grafana/latest/alerting/configure-notifications/manage-contact-points/integrations/configure-teams/
- https://grafana.com/docs/grafana/latest/alerting/set-up/provision-alerting-resources/terraform-provisioning/
- https://grafana.com/docs/grafana/latest/datasources/azure-monitor/alerting/
- https://github.com/grafana/terraform-provider-grafana/blob/v4.55.0/docs/resources/rule_group.md
- https://github.com/grafana/terraform-provider-grafana/blob/v4.55.0/docs/resources/contact_point.md
- https://learn.microsoft.com/en-us/azure/managed-grafana/how-to-use-azure-monitor-alerts
- https://learn.microsoft.com/en-us/azure/managed-grafana/known-limitations
- https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-rule-groups
- https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-pricing
- https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- https://prometheus.io/docs/prometheus/latest/querying/basics/#staleness
- https://prometheus.io/docs/prometheus/latest/querying/functions/#increase
- https://grafana.com/docs/grafana/latest/alerting/fundamentals/alert-rules/annotation-label/
- https://grafana.com/docs/grafana/latest/alerting/monitor/
- https://grafana.com/docs/grafana/latest/datasources/prometheus/query-editor/
- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/prometheus-metrics-scrape-configuration
- https://kubernetes.io/docs/reference/instrumentation/metrics/
- https://github.com/Azure/prometheus-collector/blob/main/GeneratedMonitoringArtifacts/Default/recommendedMetricAlerts.json
- https://github.com/Azure/prometheus-collector/blob/main/AddonBicepTemplate/recommendedMetricAlerts.bicep
- https://github.com/Azure/prometheus-collector/blob/main/otelcollector/shared/configmap/mp/testdata/default-linux-ds.yaml
- https://github.com/Azure/prometheus-collector/blob/main/mixins/kubernetes/alerts/storage_alerts.libsonnet
