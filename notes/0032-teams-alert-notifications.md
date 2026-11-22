# 0032 – Alert notifications to Teams: action group → Logic App → Workflows webhook

## Context

Alerts are evaluated in Azure Monitor (0031), and they need to reach people. The playground stands in for a real project, so the channel follows the real-project rule: severity decides the channel. "Pages and tickets are the only valid ways to get a human to take action" (SRE workbook). "Email alerts are of very limited value and tend to easily become overrun with noise" (SRE book). Non-urgent alerts go to a Teams channel the team watches. Paging (on-call) isn't set up yet.

## How it works

**Path:** alert rule → action group `ag-playground-teams` → Logic App `logic-playground-teams-alerts` → Teams Workflows webhook → channel. Everything is in `infra/cloud/teams-alerts.tf` except the Teams side.

1. **Action group.** One `logic_app_receiver` with `use_common_alert_schema = true`, so every alert type arrives in the same shape (`data.essentials` + `data.alertContext`).
2. **Trigger `alert`** ("When a HTTP request is received"). It's the endpoint the action group calls: "The Request trigger creates a manually callable endpoint… When the caller sends a request to this endpoint, the Request trigger fires and runs the workflow." The action group gets its signed callback URL from Terraform.
   - The `schema` is the common alert schema, copied from the alerts-logic-apps page ("Paste the common alert schema into the Request Body JSON Schema field"). It only documents the payload and gives the portal designer its tokens ("The designer uses this schema to generate tokens"). azurerm requires it.
   - `alertContext` is declared empty because its content differs per alert type. The actions read fields with expressions, not designer tokens.
   - **Schema Validation stays off.** When on, a body that doesn't match the schema gets "HTTP 400 Bad Request" and the workflow doesn't run, so that alert never reaches Teams. The schema has no `required` fields, so validation would only catch wrong types (e.g. a `null` where `string` is declared). Which fields can be `null` isn't documented. The only caller is the action group, so there's nothing to filter.
3. **Action `get-webhook-url`.** An HTTP GET to Key Vault (`/secrets/teams-alerts-webhook-url?api-version=2025-07-01`) with the Logic App's system-assigned identity (`authentication.type = ManagedServiceIdentity`, audience `https://vault.azure.net`). `secureData` hides its outputs in run history.
4. **Action `post-to-teams`.** An HTTP POST of an Adaptive Card to the URL from step 3 (`@body('get-webhook-url')?['value']`). `secureData` hides its inputs (URL and card) in run history.
   - Both are `azurerm_logic_app_action_custom`, because `azurerm_logic_app_action_http` has no `runtimeConfiguration`.

**The card.** The envelope (`type: message`, `attachments`, `contentType: application/vnd.microsoft.card.adaptive`, `AdaptiveCard` version 1.2) follows the example in the Teams incoming webhook doc. Content:

- **Title:** `<monitorCondition>: <alertRule>`, e.g. "Fired: KubePodCrashLooping".
- **Description:** `coalesce(alertContext.annotations.description, essentials.description)`.
  - Prometheus rules carry their text in annotations. The schema doc's Prometheus sample has `alertContext.annotations`, and our rules name the annotation `description`.
  - Other alert types have no `annotations`. The `?` operator then yields `null`, and `coalesce` ("Return the first non-null value") falls back to `essentials.description` ("The description, as defined in the alert rule").
  - An empty string doesn't fall back: "Empty strings… aren't null".
- **Facts:** severity, fired, resolved, targets (`alertTargetIDs`).
- Field choice: the alerts-logic-apps examples use `alertRule`, `severity`, `firedDateTime` (Teams tab) and `monitorCondition`, `alertTargetIDs` (email tab). Microsoft calls these "For example". There's no prescribed set. `resolvedDateTime` and `description` were added from the common schema's essentials.
- `size: medium`, `weight: bolder` and `wrap: true` were chosen for readability. They're valid in the Adaptive Card 1.2 schema. `wrap` defaults to `false` ("Otherwise, text is clipped"), which would cut long rule names.
- `@{…}` is Logic Apps string interpolation ("The result is always a string").

**Teams side, created by hand.**

- Office 365 connectors (old incoming webhooks) "were retired in May 2026". Workflows is the replacement.
- In the channel: Workflows → "Send webhook alerts to a channel" → copy the URL.
- Add a co-owner: "Workflows are linked only to specific users… can become orphan flows in the absence of an owner if no co-owners assigned."
- Limits: 28 KB per message, four requests per second. The HTTP action's default retry ("up to 4 retries at exponentially increasing intervals", on 408/429/5xx) covers bursts.

## Decision

**Logic App between the action group and Teams.** Azure has no Teams action type. A Workflows webhook only accepts "Adaptive Card and MessageCard payloads", and Azure sends the common alert schema. Microsoft: "If the webhook endpoint expects a specific schema, for example, the Microsoft Teams schema, use the Logic Apps action type to manipulate the alert schema." Each piece is documented, but no Microsoft page shows this whole chain.

Rejected:

- **The Logic Apps Teams connector** ("Post message in a chat or channel", the example in the alerts docs). It needs an interactive sign-in ("Sign in to Teams when you're prompted"), posts as that user ("Select User from the Post as dropdown"), the connection "is not shareable", and "Posting a message or adaptive card to a private channel currently isn't supported." The Logic App here makes only plain HTTP calls, so it's fully Terraform.
- **Action group webhook straight to the Teams URL.** Wrong payload format.

**The webhook URL is a secret** (anyone with it can post). It takes the Cloudflare token's path up to CI (0015), then goes into Key Vault:

1. By hand into `infra/init/secrets.auto.tfvars` as `teams_alerts_webhook_url`. `bootstrap.sh` fails if it's missing.
2. Init writes the GitHub org secret `TEAMS_ALERTS_WEBHOOK_URL`, visible to `infra` only (`infra/init/main.tf`).
3. CI passes it as `TF_VAR_teams_alerts_webhook_url` to Plan and Apply (`infra/.github/workflows/deploy.yml`). The variable is `ephemeral`, which omits it "from state and plan files". A plain sensitive variable is still stored in state. A validation rejects anything that isn't `https://`.
4. Terraform writes it into Key Vault with `value_wo`, which is write-only and not persisted to state or plan.
5. The Logic App reads it at runtime (steps 3–4 above).

Why Key Vault and not only a `securestring` workflow parameter:

- Well-Architected: "Use a secret management system, like Azure Key Vault, to store secrets", and "Don't store keys and secrets for any environment type in application configuration files or … CI/CD pipelines."
- A `securestring` parameter isn't a store. It's write-only ("aren't returned with the resource definition and aren't accessible when viewing the resource after deployment"), but its value has to come from somewhere. Microsoft's own pattern feeds it "from Azure Key Vault at deployment".
- The URL still passes through a GitHub secret, which bends the CI/CD rule above. That's accepted so a rebuild runs end to end with no manual step after init. A real project would enter the value into Key Vault by hand.

**Own vault, `kv-alerts-<suffix>`, not the app's `kv-playground-…`.** Microsoft's guidance:

- "Our recommendation is to use a vault per application per environment (Development, Pre-Production, and Production) with roles assigned at the key vault scope." (Key Vault RBAC guide)
- "Assigning roles on individual keys, secrets and certificates is not recommended." The exceptions are individual user access and secrets shared between applications.
- "Grouping secrets into the same vault increases the blast radius of a security event because attacks might be able to access secrets across concerns." (Key Vault security best practices)

In a shared vault, the Logic App would either read the backend's secrets (vault scope) or need per-secret roles (not recommended). With its own vault it gets `Key Vault Secrets User` at vault scope. Terraform (the CI identity) gets `Key Vault Secrets Officer` there to write the secret.

## Consequences

- **Changing the URL:** update tfvars, run init, then bump `value_wo_version` in `teams-alerts.tf`. "Terraform has no way of knowing if a write-only argument value has changed", so without the bump CI does nothing.
- **No grouping.** Each alert becomes its own card. Action groups don't group, and Grafana's notification policies aren't in this path (0031).
- **The Teams workflow belongs to people,** not to the team. Without a co-owner it can be orphaned and stop posting.
- **Verified** (2026-11-22):
  - CI run 45003455703 created the resources: `Apply complete! Resources: 14 added, 0 changed, 0 destroyed.` (with the 0033 rules).
  - `az monitor action-group test-notifications create … --alert-type metricstaticthreshold` returned `LogicApp: Succeeded`.
  - Run at 08:41:18 UTC: `get-webhook-url` Succeeded (OK), `post-to-teams` Succeeded (202 Accepted). So the identity can read the vault, and a GET without a version returns the current secret.
  - Run history: no link to `get-webhook-url` outputs and none to `post-to-teams` inputs. The remaining links don't contain the webhook host.
  - The test payload had all declared `string` fields as strings, no `resolvedDateTime` (absent, not `null`), and an extra `configurationItems`. Its `alertContext` had `properties`, `conditionType`, `condition` and no `annotations`.
- **Not verified yet:**
  - A real Prometheus payload: does `alertContext.annotations.description` arrive as in the doc sample?
  - The card's look in the channel. The API only shows that Teams accepted it.
  - Whether `apply tfplan` actually needs the ephemeral variable again. The docs don't say. CI passes it to both steps, so this can't be told from the run.

## References

- https://sre.google/workbook/alerting-on-slos/
- https://sre.google/sre-book/monitoring-distributed-systems/
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-logic-apps
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-common-schema
- https://learn.microsoft.com/en-us/azure/connectors/connectors-native-reqres
- https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook
- https://learn.microsoft.com/en-us/connectors/teams/
- https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app
- https://learn.microsoft.com/en-us/azure/logic-apps/authenticate-with-managed-identity
- https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-exception-handling
- https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference
- https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language
- https://github.com/microsoft/AdaptiveCards/blob/main/schemas/1.2.0/adaptive-card.json
- https://learn.microsoft.com/en-us/rest/api/keyvault/secrets/get-secret/get-secret
- https://learn.microsoft.com/en-us/azure/well-architected/security/application-secrets
- https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide
- https://learn.microsoft.com/en-us/azure/key-vault/general/best-practices
- https://developer.hashicorp.com/terraform/language/manage-sensitive-data/write-only
- https://developer.hashicorp.com/terraform/language/values/variables
- https://github.com/hashicorp/terraform-provider-azurerm/blob/v5.0.1/internal/services/logic/logic_app_workflow_resource.go
