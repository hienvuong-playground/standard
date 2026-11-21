# 0021 – Azure Managed Grafana next to the self-hosted one

## Context

Evaluating Azure Managed Grafana (AMG). The self-hosted Grafana in AKS stays as it is. The two run in parallel until the evaluation is done, then AMG is removed. Dashboards and datasources on AMG will later be managed with Terraform.

## Decision

**One Standard instance, `amg-playground`,** in `infra/cloud/managed-grafana.tf`. Grafana 13, system-assigned identity, the Azure Monitor workspace attached through `azure_monitor_workspace_integrations` (adds a Prometheus datasource). `sku` isn't set: it defaults to `Standard`, and new Essential workspaces can no longer be created. The pinned azurerm 5.0.1 only accepts major versions `12` and `13` (checked with `terraform validate`).

**Its identity can read everything in the subscription.** This is not least privilege, on purpose: it's a test, and missing permissions shouldn't get in the way.

- `Monitoring Reader` on the subscription: Azure Monitor metrics, Log Analytics / App Insights logs, and resource reads. It includes everything `Reader` has (`*/read`) plus log search, and it's what the portal picks by default when you create AMG. So it was chosen over plain `Reader`.
- `Monitoring Data Reader` on the subscription: querying Prometheus in an Azure Monitor workspace is a data action, which neither `Reader` nor `Monitoring Reader` cover. The portal sets this up for you; Terraform does not, so it's assigned explicitly.

Rejected: `Contributor`. Grafana only reads, and write access adds nothing.

**Who can sign in: `Grafana Admin` for two principals.** The `Hien Test Group Admin` group (same group as the Postgres Entra admin, 0017), and the principal Terraform runs as (`data.azurerm_client_config.current`, i.e. the CI identity). The Grafana roles (`Admin` / `Editor` / `Viewer`) are data actions, so being `Owner` on the subscription does not let you into Grafana or its API. Without the role, Terraform couldn't create dashboards or datasources.

## Consequences

- Standard bills per instance around the clock. Unlike AKS and Postgres (0019), AMG can't be stopped at night.
- The URL is AMG's own `*.grafana.azure.com` endpoint (shown on the resource in the portal), not `grafana.<zone>` through Cloudflare like the self-hosted one.
- The endpoint isn't a Terraform output yet. Add one to `infra/cloud/output.tf` once another stack (e.g. the Grafana Terraform provider) needs it.
- Terraform resource names are prefixed `managed_grafana_`, so they don't get mixed up with the self-hosted `grafana_*` ones in `infra/cloud/grafana-auth.tf`.
- Verified: the CI apply succeeded, members of the admin group can sign in, and both the Azure Monitor and the Prometheus datasources return data (0023, 0024).

## References

- https://learn.microsoft.com/en-us/azure/managed-grafana/overview
- https://learn.microsoft.com/en-us/azure/managed-grafana/how-to-permissions
- https://learn.microsoft.com/en-us/azure/managed-grafana/concept-role-based-access-control
- https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/monitor
- https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/dashboard_grafana
