# 0031 – Alerts are evaluated in Azure Monitor, not in Grafana

## Context

No alert rules exist yet. 0028 planned to alert in Grafana (AMG, 0021), mainly to avoid vendor lock-in. Before building anything, the two options were compared on lock-in, availability and notification features.

## Decision

**Alert rules, notifications and suppression live in Azure Monitor:** Prometheus rule groups, metric alerts, log search alerts, activity log alerts, action groups and alert processing rules. Grafana stays the place to look at data and investigate. Grafana alerting may be added later as a parallel path to evaluate (CLAUDE.md), but not as the primary one.

Why:

- **Grafana doesn't reduce lock-in.** Lock-in is in the data, not in where the rule runs. Metrics are in the Azure Monitor workspace (PromQL) and logs in Log Analytics (KQL). A Grafana rule on Log Analytics is still KQL against Azure. PromQL is portable either way. AMG is an Azure service too. The only part that would be portable is the notification config (contact points, policies).
- **Alert where the data is, and keep extra hops out of the alert path.** With Grafana, the path is data → AMG → SMTP server → inbox, and every hop can fail silently. In Azure Monitor the alert rule is evaluated by the platform that holds the data.
  - The SLA covers the alert path itself. Azure Monitor has two 99.9% SLAs: alert rules (a minute is down if evaluation doesn't succeed "within five minutes from scheduled Alert Rule start time") and notification delivery (action groups). AMG's 99.9% only covers "HTTP requests to perform operations" and says nothing about alert evaluation (Online Services SLA, Nov 2026).
  - In AMG, "alert rules are processed in your workspace, where they share the same compute resources and query-throttling limits with dashboard rendering."
  - AMG runs "two servers… for high availability", but no doc says alerting runs in HA across them. Self-managed Grafana needs explicit HA config for that (`ha_peers` or Redis).
  - AMG zone redundancy can only be set at creation, costs extra, and isn't offered in West Europe (where we are, `infra/cloud/locals.tf`).
  - Grafana email needs our own SMTP server (prerequisite: "An SMTP server"). Action group email needs nothing.
  - The same principle holds for a self-hosted stack (OTel → Prometheus → Grafana). Prometheus's standard design evaluates rules where the data is and leaves notifications to Alertmanager: "Alerting rules in Prometheus servers send alerts to an Alertmanager", which handles "silencing, inhibition, aggregation and sending out notifications". Grafana is not part of that path.
- **Microsoft doesn't support it.** "Using Grafana alerts with an Azure Monitor service isn't officially supported by Microsoft", and "for any Azure Monitor service, including … Prometheus, define and manage your alert rules in Azure Monitor." This is a support statement, not a technical limit: Grafana's own docs say all Azure Monitor query types support alerting.
- **Some alerts only exist in Azure.** Service Health and Resource Health are activity log alerts, on Microsoft's reliability checklist, and free. Grafana has no equivalent.
- **One system instead of two.** Even with Grafana alerting, Service Health and "is AMG itself down?" would still need Azure Monitor.

Rejected: Grafana alerting as the primary path. Not because Grafana can't alert (Grafana Alerting is a full feature), but for the reasons above.

## Where the alert list comes from

Not invented here. One standard source per layer:

- **Azure resources and platform: Azure Monitor Baseline Alerts (AMBA).** Per resource type (Postgres, Key Vault, ACR…), with metric, threshold and severity. "Enabled" means "Must Have alert". It also covers Service Health and Resource Health. AMBA is "based on Microsoft's recommendations and field experience", from the `Azure` GitHub org.
  - Thresholds are "intended as a starting point… You may need to adjust these thresholds". Some don't fit as-is, e.g. ACR `SuccessfulPullCount > 50` at Sev2 isn't actionable.
  - AMBA deploys through Azure Policy. Here the definitions are ported to Terraform instead, like the AddonTerraformTemplate in 0028.
  - The Service Bus, Storage and Log Analytics pages came back empty when checked. Look again before porting those.
- **Cluster: Microsoft's recommended Kubernetes alerts.** Prometheus rules (30 in the template, 29 on the Learn page) and 2 platform metric rules, listed in 0028. AMBA's AKS page only has platform metrics.
- **Application: golden signals, later SLO burn rate.** AMBA has nothing here. This is the layer that should page: "it's better to spend much more effort on catching symptoms than causes" (Google SRE book). Infrastructure alerts are for diagnosis: "Infrastructure alerts tend to be noisy and are often ignored when treated like paging signals" (Grafana best practices).

## Implementation

- Notifications go to a Teams channel through action group → Logic App → Teams Workflows webhook: 0032.
- The first rules are Microsoft's recommended Kubernetes alerts (30 Prometheus + 2 platform metric rules): 0033.

## Consequences

- **We give up Grafana's notification features.** No native Slack or Teams (Teams goes through a Logic App, 0032), no grouping of several alerts into one notification (built for Prometheus alerts with Logic Apps Batch, 0034), no Go templates for the message.
- **Limits.** Email: at most 100 per hour per address per region. SMS and voice: one every five minutes.
- **The nightly AKS/Postgres stop (0019) may need an alert processing rule** with a recurring window. Prometheus rules don't fire while the cluster is stopped: an alert is only active when the expression "results in one or more vector elements", and a stopped cluster sends no samples. The risk is the start: 0029 saw seven containers crash when AKS started, which restart and crash-loop rules can catch. Platform metric alerts and Resource Health on stopped resources aren't checked yet (0033). Processing rules don't affect Service Health alerts.
- **Not verified:** how Azure Monitor alerts behave in a regional outage (no reliability page found), and the price of Prometheus rule groups.
- 0028's plan to alert in Grafana is superseded by this note.

## References

- https://learn.microsoft.com/en-us/azure/managed-grafana/how-to-use-azure-monitor-alerts
- https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services
- https://learn.microsoft.com/en-us/azure/reliability/reliability-managed-grafana
- https://learn.microsoft.com/en-us/azure/managed-grafana/how-to-smtp-settings
- https://grafana.com/docs/grafana/latest/alerting/set-up/configure-high-availability/
- https://prometheus.io/docs/alerting/latest/overview/
- https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- https://grafana.com/docs/grafana/latest/datasources/azure-monitor/alerting/
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/best-practices-alerts
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-processing-rules
- https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-logic-apps
- https://grafana.com/docs/grafana/latest/alerting/configure-notifications/manage-contact-points/
- https://azure.github.io/azure-monitor-baseline-alerts/services/
- https://azure.github.io/azure-monitor-baseline-alerts/patterns/alz/Resources/FAQ/
- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-metric-alerts
- https://sre.google/sre-book/monitoring-distributed-systems/
- https://sre.google/workbook/alerting-on-slos/
- https://grafana.com/docs/grafana/latest/alerting/guides/best-practices/
