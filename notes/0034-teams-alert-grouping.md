# 0034 – Grouping Prometheus alerts into one Teams card per rule and cluster

## Context

After a node restart, 63 `KubePodContainerRestart` alerts gave 63 Fired and 63 Resolved cards (0033). Teams then took ~13 minutes to post them (0032). These weren't duplicates: each was a different series ("No duplicates", 0033). Azure already deduplicates: stateful alerts "will not fire again or trigger any more actions until the conditions are resolved" (alerts overview). What's missing is **grouping**: many different alerts with one cause in one notification.

Constraints: the rule stays (Microsoft's recommended set, 0033), and Teams stays the channel (0032).

## Decision

**Copy Alertmanager's grouping and Teams template into a Logic App (Consumption) that uses the built-in Batch trigger.**

- Alertmanager is the standard tool for this: "Grouping categorizes alerts of similar nature into a single notification… hundreds to thousands of alerts may be firing simultaneously". Its example groups "by their cluster and alertname".
- Batch is a built-in Logic Apps feature that "collects messages into a *batch* and waits until your specified criteria are met before releasing" them. It needs no extra service.

Rejected:

- **Run Alertmanager itself.** Inside the cluster it dies with the cluster, and the cluster stops every night (0019). Outside the cluster:
  - It expects alerts to be re-sent: "Alerts themselves are **not** persisted - Prometheus re-sends firing alerts regularly". Azure sends a state change once, so a restart would lose every firing alert.
  - HA gossip needs "Both TCP and UDP", but Container Apps ingress supports only "HTTP and TCP".
- **Incident platform** (PagerDuty etc.). Teams is the decided channel.
- **Logic App Standard or an integration account** to run JavaScript for the template. Standard costs a plan for what is only a message. Inline JavaScript in Consumption needs "Link to an integration account".
- **Own batching with a queue and timer.** That would rebuild Alertmanager's grouping by hand. Batch already does it.
- **Disable `KubePodContainerRestart`.** It's part of Microsoft's recommended set, and it catches real restarts in a real cluster.

## How it works

Everything is in `infra/cloud/teams-alerts.tf`.

**Routing, in the existing workflow** (`logic-playground-teams-alerts`):

- Action `is-prometheus` checks `essentials.monitoringService == 'Prometheus'`. Azure sets this field per rule type. Values seen: `Prometheus`, `Platform` (metric alert), `ServiceHealth`.
- If true: `SendToBatch` sends the alert's `data` to the batch workflow, with partition `alertname|cluster`, then `Terminate` (Succeeded) ends the run.
- Otherwise, the single-card path runs unchanged (0032). It runs after any status of `is-prometheus`, because the If action's status for a false condition isn't documented. A failed `send-to-batch` therefore also falls back to a single card.
- Why only Prometheus: the template is built on labels. Metric alerts and Service Health alerts have no `alertContext.labels`.

**Batch workflow** (`logic-playground-teams-alerts-batch`):

- Trigger `Batch`, `mode: Inline`, release every minute (the minimum, "60 seconds or 1 minute"). Alertmanager's default `group_wait` is 30s.
- The JSON shape comes from Microsoft's own sources, because the schema reference doesn't document it:
  - `BatchTriggerInputs` and `SendToBatchActionInputs` in the LogicAppsUX designer source.
  - A Consumption workflow in Azure-Sentinel (Group-IB playbooks) that uses both, with no integration account. Items are read as `triggerBody()['items']` → `item()['content']`.
- One card per distinct `alertname|cluster` in the released items, so it works whether a release holds one partition or several.

**The card follows Alertmanager's msteamsv2 notifier:**

- **Title** is `__subject`: `[FIRING:<n firing>]` or `[RESOLVED]`, the group label values, and `(<common labels minus group labels>)`.
  - Example: `[FIRING:63] KubePodContainerRestart aks-playground (warning)`.
  - Status is "firing if at least one alert is firing, otherwise resolved".
- **Text** is `msteamsv2.default.text`: `# Alerts Firing:` and `# Alerts Resolved:`, each only when not empty.
  - Every alert gets `Labels:` and `Annotations:` as `  - name = value`, sorted by name (`__text_alert_list_markdown`).
- **Card layout** is from `notify/msteamsv2/msteamsv2.go`:
  - AdaptiveCard 1.2.
  - The title is a TextBlock, Bolder, Medium, wrap, style `heading`, colour `Attention` (firing) or `Good` (resolved).
  - The text is a TextBlock with wrap, and `msteams.width = full`.
- **Listing labels.** Workflow expressions have no `keys()` function. `xml()` turns the labels object into `<r><name>value</name>…</r>`, and `xpath()` reads `name()` and `string()` of each element by position. Prometheus label names "SHOULD match the regex `[a-zA-Z_][a-zA-Z0-9_]*`", which are valid XML names.

**Where the card differs from the template, and why:**

- **No `Source:` line.** The common alert schema has no GeneratorURL.
- **At most 50 alerts per card, then `… N alerts truncated`.**
  - The full template for the 63 real alerts measured 26,308 bytes. Teams: "When the size exceeds 28 KB, you receive an error".
  - msteamsv2 has no limit of its own. The cap copies Alertmanager's webhook `max_alerts` ("Alerts above this threshold are truncated") and `truncatedAlerts`.
  - The wording of the line is ours, because the template has none.
- **`#` headers stay**, as in the template, even though Teams Adaptive Cards don't support headers. They show as plain text.

## Consequences

- **Other alert types** (resource metric alerts from AMBA, Service Health, log search alerts) still get one card each. Their `alertContext.condition.allOf[].dimensions` are name/value pairs like labels, so they can be added to the batch path if they ever burst.
- **Application alerts** on OTel metrics are Prometheus rules on the AMW (0022), so they're grouped too.
- **Differences from Alertmanager:**
  - Batch releases on a fixed clock, not 30s after the first alert. A burst that crosses a minute boundary gives two cards.
  - There is no state per group. A card only lists alerts that changed in that minute, with no `group_interval` re-send of the whole group and no `repeat_interval`.
  - There is no inhibition. Silences are alert processing rules (0031).
- **Cost.**
  - "Consumption Built-in Actions" in westeurope: first 4,000 free, then $0.000025 each (Azure Retail Prices API).
  - The trigger at one run per minute is at most ~43,200 runs a month, about $1, if empty minutes are billed.
  - A card for 50 alerts is roughly 700 actions, about $0.02.
- **Not verified yet** (check in run history on the first cluster start after deploy):
  - Whether one release holds one partition or several.
  - Whether the trigger runs (and bills) on empty minutes.
  - That `xpath()` returns plain strings for `name()` and `string()`.
  - That `union()` of the keys with itself removes duplicates within one array (the doc example only shows duplicates across two arrays).
  - That an If with a false condition reports Succeeded.
  - The exact newlines Teams renders.

## References

- https://prometheus.io/docs/alerting/latest/alertmanager/
- https://prometheus.io/docs/alerting/latest/configuration/
- https://prometheus.io/docs/alerting/latest/notifications/
- https://prometheus.io/docs/alerting/latest/high_availability/
- https://prometheus.io/docs/concepts/data_model/
- https://github.com/prometheus/alertmanager/blob/main/template/default.tmpl
- https://github.com/prometheus/alertmanager/blob/main/notify/msteamsv2/msteamsv2.go
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-common-schema
- https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-batch-process-send-receive-messages
- https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-actions-triggers
- https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference
- https://learn.microsoft.com/en-us/azure/logic-apps/add-run-javascript
- https://learn.microsoft.com/en-us/azure/container-apps/ingress-overview
- https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook
- https://learn.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/cards-format
- https://github.com/Azure/LogicAppsUX/blob/main/libs/logic-apps-shared/src/utils/src/lib/models/logicAppsV2.ts
- https://github.com/Azure/Azure-Sentinel/tree/master/Solutions/Group-IB/Playbooks
- https://prices.azure.com/api/retail/prices
