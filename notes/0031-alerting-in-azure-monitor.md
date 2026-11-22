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

## How alerts reach Teams

Severity decides the channel: "pages and tickets are the only valid ways to get a human to take action" (SRE workbook), and "Email alerts are of very limited value" (SRE book). Non-urgent alerts go to a Teams channel. Paging (on-call) isn't set up yet.

**Path:** alert rule → action group `ag-playground-teams` → Logic App `logic-playground-teams-alerts` → Teams Workflows webhook. All in `infra/cloud/teams-alerts.tf`.

- **Why a Logic App in between.** Azure has no Teams action type. A Workflows webhook only takes "Adaptive Card and MessageCard payloads", and Azure sends the common alert schema. Microsoft: "If the webhook endpoint expects a specific schema, for example, the Microsoft Teams schema, use the Logic Apps action type". The Logic App turns the alert into an Adaptive Card. Prometheus rules put their text in `alertContext.annotations`, so the card shows `annotations.description`, falling back to `essentials.description`.
- **Rejected: the Logic Apps Teams connector** ("Post message in a chat or channel", the example in the alerts docs). It needs an interactive sign-in ("Sign in to Teams when you're prompted"), posts as that user, the connection "is not shareable", and private channels aren't supported. The Logic App here only makes plain HTTP calls, so it's fully Terraform.
- **Rejected: action group webhook straight to Teams.** Wrong payload format, see above.
- **Teams side, created by hand.** Office 365 connectors (old incoming webhooks) "were retired in May 2026". Workflows is the replacement. In the channel: Workflows → "Send webhook alerts to a channel" → copy the URL. Add a co-owner: "Workflows are linked only to specific users… can become orphan flows in the absence of an owner if no co-owners assigned." Limits: 28 KB per message, four requests per second. The HTTP action's default retry ("up to 4 retries at exponentially increasing intervals", on 408/429/5xx) covers bursts.

**The webhook URL is a secret** (anyone with it can post). Its path, the same as the Cloudflare token's up to CI (0015):

1. By hand into `infra/init/secrets.auto.tfvars` as `teams_alerts_webhook_url`. `bootstrap.sh` fails if it's missing.
2. Init writes the GitHub org secret `TEAMS_ALERTS_WEBHOOK_URL`, visible to `infra` only.
3. CI passes it as `TF_VAR_teams_alerts_webhook_url`, an `ephemeral` variable. Ephemeral omits it "from state and plan files"; a normal sensitive variable still lands in state.
4. Terraform writes it into Key Vault with `value_wo` (write-only, not persisted to state or plan).
5. At runtime the Logic App reads it with its system-assigned identity (HTTP GET to Key Vault, audience `https://vault.azure.net`) and posts to it. Both HTTP actions use `secureData`, so the URL doesn't show in run history.

Why Key Vault and not only a `securestring` workflow parameter:

- Well-Architected: "Use a secret management system, like Azure Key Vault, to store secrets", and "Don't store keys and secrets for any environment type in application configuration files or … CI/CD pipelines."
- A `securestring` parameter isn't a store. It's "not returned with the resource definition", but its value comes from somewhere, and Microsoft's own pattern feeds it "from Azure Key Vault at deployment".
- It still passes through a GitHub secret, which bends the CI/CD rule above. Accepted so a rebuild runs end to end with no manual step after init. A real project would enter the value into Key Vault by hand instead.

**Own vault, `kv-alerts-<suffix>`, not the app's `kv-playground-…`.** Microsoft's guidance:

- "Our recommendation is to use a vault per application per environment (Development, Pre-Production, and Production) with roles assigned at the key vault scope." (Key Vault RBAC guide)
- "Assigning roles on individual keys, secrets and certificates is not recommended." The exceptions are individual user access and secrets shared between applications.
- "Grouping secrets into the same vault increases the blast radius of a security event because attacks might be able to access secrets across concerns." (Key Vault security best practices)

In a shared vault, the Logic App would either read the backend's secrets (vault scope) or need per-secret roles (not recommended). With its own vault it gets `Key Vault Secrets User` at vault scope. Terraform gets `Key Vault Secrets Officer` there to write the secret.

**Changing the URL:** update tfvars, run init, then bump `value_wo_version` in `teams-alerts.tf`. "Terraform has no way of knowing if a write-only argument value has changed", so without the bump CI does nothing.

**Cluster rules** are in `infra/cloud/kubernetes-alerts.tf`: Microsoft's recommended rules from the template behind https://aka.ms/azureprometheus-recommendedmetricalerts. That's 30 rules (11 cluster, 3 node, 16 pod). The Learn page lists 29, missing `KubeNodeDiskUsageHigh`. The file is generated from the template JSON. All 30 expressions, `for`, severity, descriptions and resolve times were checked against it. The 3 ResourceQuota rules never fire here, because the cluster has no ResourceQuota. They're kept unchanged, like the recording rules in 0028.

The same page lists 2 **platform metric rules**: node CPU > 95% and node memory working set > 100%, both over 5 minutes. Microsoft's ARM and Bicep templates don't include them; the page only enables them with a portal toggle. Their full definitions come from AMBA (`services/ContainerService/managedClusters/alerts.yaml`):

- Metrics `node_cpu_usage_percentage` and `node_memory_working_set_percentage`.
- Average, GreaterThan, window and frequency PT5M, severity 3.

These match the page's thresholds and timeframe. They're in the same file as `azurerm_monitor_metric_alert`.

- AMBA marks both `verified: false`.
- There's no split by the `node` dimension, so the alert looks at the metric over the whole cluster ("Aggregated average CPU utilization measured in percentage across the cluster"), not per node.
- How metric alerts behave while AKS is stopped (no data) isn't checked.

## Consequences

- **We give up Grafana's notification features.** No native Slack or Teams (Teams goes through the Logic App above), no grouping of several alerts into one notification, no Go templates for the message.
- **Limits.** Email: at most 100 per hour per address per region. SMS and voice: one every five minutes.
- **The nightly AKS/Postgres stop (0019) may need an alert processing rule** with a recurring window. Prometheus rules don't fire while the cluster is stopped: an alert is only active when the expression "results in one or more vector elements", and a stopped cluster sends no samples. The risk is the start: 0029 saw seven containers crash when AKS started, which restart and crash-loop rules can catch. Platform metric alerts (Postgres) and Resource Health on stopped resources aren't checked yet. Processing rules don't affect Service Health alerts.
- **Not verified:** how Azure Monitor alerts behave in a regional outage (no reliability page found), and the price of Prometheus rule groups.
- **To confirm on the first CI run:**
  - Whether `terraform apply tfplan` needs the ephemeral variable again. The docs don't say, so CI passes it to both Plan and Apply.
  - That the Key Vault GET without a version returns the latest one.
  - That Prometheus alerts carry `annotations` in the payload as in the schema doc's sample.
  - Then: **Test** on the action group, and one real alert from a crash-looping pod.
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
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-common-schema
- https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook
- https://learn.microsoft.com/en-us/connectors/teams/
- https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app
- https://learn.microsoft.com/en-us/azure/logic-apps/authenticate-with-managed-identity
- https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-exception-handling
- https://learn.microsoft.com/en-us/rest/api/keyvault/secrets/get-secret/get-secret
- https://learn.microsoft.com/en-us/azure/well-architected/security/application-secrets
- https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide
- https://learn.microsoft.com/en-us/azure/key-vault/general/best-practices
- https://developer.hashicorp.com/terraform/language/manage-sensitive-data/write-only
- https://developer.hashicorp.com/terraform/language/values/variables
- https://github.com/Azure/prometheus-collector/blob/main/GeneratedMonitoringArtifacts/Default/recommendedMetricAlerts.json
- https://github.com/Azure/azure-monitor-baseline-alerts/blob/main/services/ContainerService/managedClusters/alerts.yaml
- https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-containerservice-managedclusters-metrics
