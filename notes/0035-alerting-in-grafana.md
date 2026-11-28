# 0035 – Alerts move to Grafana (AMG), Azure Monitor alerts disabled

## Context

0031 put alerting in Azure Monitor. In practice two things hurt: Azure has no grouping (0034 rebuilt Alertmanager's grouping in a Logic App), and the Logic App workflow code is hard to read. The extra Azure Monitor SLA on the alert path (0031) was judged not worth that. Grafana's Alertmanager groups natively.

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
- **If the newest samples aren't ingested yet:** an instant query takes "the newest sample that is less than the lookback period ago" (5 minutes), so it uses a slightly older sample, not nothing. Only after more than 5 minutes without data do the series disappear. Then rules are Normal, and firing alerts resolve (`missing_series_evals_to_resolve = 1`), as in Prometheus. That's a broken metrics pipeline, which these rules (and the unreachable rule) don't detect. The real AMW ingestion delay hasn't been measured.

**No Data = OK.** For these rules an empty result means healthy: the query only returns series that are in trouble. Grafana's default No Data handling assumes the opposite (the query returns numbers when healthy, so empty = broken) and would create a `DatasourceNoData` alert. Here "healthy" and "no data at all" look the same, so No Data can't tell them apart.

- Rewriting the rules Grafana-style (threshold as its own node, No Data default) was rejected. 11 of 30 rules use the comparison as a filter inside `and`/`unless`/`or`/`changes()`/`!=`, so the logic would have to be rewritten. Several metrics only exist with their objects (no ResourceQuota in the cluster, so the quota rules would be NoData forever). The nightly stop (0019) would raise NoData daily. The missing-data guide itself suggests Normal for "intermittent data".
- Grafana-style rules with the default No Data are the right choice for rules written here from scratch (e.g. application golden signals), where the query always returns data when healthy.
- So these rules don't detect "the cluster is gone". That needs its own rule, not added yet.

**Query errors: OK on the 30 rules, one dedicated rule.** Grafana's connectivity-errors guide: "A single data source error can lead to multiple alerts firing simultaneously", so "Create a separate alert rule dedicated to detecting when the data source itself is unreachable". `AzureMonitorWorkspaceUnreachable` queries `vector(1)` with error and no-data handling set to Alerting, `for = 5m`.

**2 platform metric rules (Platform group):** node CPU > 95 % and memory working set > 100 %. They're part of Microsoft's recommended set (0033), but they're Azure platform metrics of the AKS resource, not Prometheus data, so they go through the Azure Monitor data source. Query with `PT1M` grain over the last 5 minutes (here `from = 300` does matter) → Reduce mean (drop non-numeric) → threshold. That matches AMBA's "Average over PT5M", every 5 minutes. No Data = OK (AKS stopped). Errors = Error: with two rules a `DatasourceError` per rule is fine. Subscription, resource group and cluster name come from `cloud` outputs (`subscription_id`, `resource_group_name`, `aks_cluster_name`).

Both data sources were created by Azure and use managed identity (`msi`), which alerting needs: rules "run as background processes without a user context".

## Notifications

- Contact point `teams` → the same Teams Workflows webhook (0032). The Grafana docs use the template "Post to a channel when a webhook request is received". The current workflow is "Send webhook alerts to a channel". If the test card doesn't show, create a new workflow and follow 0032's path (tfvars → init → GitHub secret).
- Default policy: `group_by = ["alertname", "cluster"]` (Alertmanager's example, as in 0034). It groups by **labels** ("same exact label values for the labels configured in the `Group by` option"). Annotations are only text. Nothing here sets the two labels:
  - `alertname` is reserved: "automatically added by Grafana: `alertname`: the name of the alert rule".
  - `cluster` comes from the data. Azure appends "the last part of the cluster's resource ID… to every time series", and Grafana copies query labels onto the alert. Checked: every `up` series in the AMW has `cluster="aks-playground"`.
  - The platform rules (Azure Monitor) and `DatasourceError`/`DatasourceNoData` have no `cluster` label. They group by `alertname` only. Timers are Grafana's defaults (30s / 5m / 4h). This replaces the Logic App batch and adds what it lacked: per-group state, `group_wait` from the first alert, repeat.
- **The webhook URL is in `grafana.tfstate`.** The contact point's `url` is `Sensitive`, not write-only, so an ephemeral variable can't be used. Accepted. CI passes `TEAMS_ALERTS_WEBHOOK_URL` to the `terraform-grafana` job (`infra/.github/workflows/deploy.yml`).

## Consequences

- **Microsoft doesn't support it:** "Using Grafana alerts with an Azure Monitor service isn't officially supported by Microsoft." That's a support statement, not a technical limit.
- **Alert rules "share the same compute resources and query-throttling limits with dashboard rendering"** in AMG. The X1 limit is 500 rules. We use 33.
- **If AMG itself is down, nothing alerts and nobody is told.** Not covered yet.
- **Rules provisioned by Terraform can't be edited in the UI.** Same as dashboards (0026).
- **Every rule in AMG goes to Teams through the default policy,** including rules made by hand in the UI.
- **Teams throttling stays** (Flow bot, 25 per 300 s, 0032), but grouping keeps the card count low.
- **Data source uids are hard-coded** (`amw-playground`, `azure-monitor-oob`). Azure creates both data sources: the AMW integration (`azure_monitor_workspace_integrations`) makes `Managed_Prometheus_amw-playground` with uid = the AMW name, and AMG makes `Azure Monitor` ("oob") at creation. No doc found for that naming. If a uid changes, the `grafana_data_source` lookup fails at plan, so nothing points at the wrong source silently. Open options:
  - Manage our own data sources in Terraform (url from `amw_query_endpoint`, MSI). The rules then reference them, but AMG gets a duplicate of each unless the integration is removed, which may take other Azure-made things with it (to check).
  - Derive the uid from the AMW name. Rejected: it still relies on the undocumented naming.
- **Verified before deploy (2026-11-28):**
  - The template on `main` still matches all 30 expressions in `kubernetes-alerts.tf`.
  - The template was POSTed to Grafana's import API (`/api/convert/prometheus/config/v1/rules/convert-test`, rules paused) on AMG. The output was compared with the Terraform rules: 30/30 match on group, `for`, `keep_firing_for`, labels (minus `__converted_prometheus_rule__`), annotations, the query model, time range, math, threshold, condition, no-data/error state and missing-series setting. The only differences were `60m` vs `1h`, which the provider treats as equal. The test folder was deleted.
  - `terraform validate` for both stacks.
- **Not verified yet:** the deploy, the contact point test, a real alert end to end.

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
