# 0026 – AMG dashboards in Terraform, with an Entra token

## Context

The folder, dashboards and home page on AMG (0023, 0024, 0025) were made by hand through the Grafana API. They should be code, so a rebuild (0016) brings them back and changes go through CI. The Grafana Terraform provider needs some way to authenticate against AMG's Grafana API.

## Decision

**Auth: an Entra access token of the CI identity.** After `azure/login` (OIDC), the workflow runs `az account get-access-token --resource ce34e7e5-485f-4d76-964f-b3d2b16d1e4f`, masks the token, and puts it in `GRAFANA_AUTH` (`infra/.github/workflows/deploy.yml`, job `terraform-grafana`).

- `ce34e7e5-…` is AMG's Entra app id, the same in every tenant.
- The CI identity already has `Grafana Admin` on AMG (0021). Nothing is stored.
- The token lasts about an hour, enough for plan + apply.
- The provider only documents API keys, basic auth and service account tokens. It sends `auth` as a Bearer token, and AMG accepts an Entra token there. Checked with a local read-only plan before the first CI run.

Rejected:

- **Grafana service account token.** Needs `api_key_enabled = true` on AMG and a long-lived secret in GitHub or Key Vault. Tokens expire after a day by default, so either "no expiry" or rotation. Someone also has to create the first token, which would mean another manual step in `init`.
- **`az grafana dashboard create` from `local-exec`.** No state, no drift detection.

**A separate stack, `infra/grafana/`.** The provider URL is `azurerm_dashboard_grafana.main.endpoint`. In the `cloud` stack that value is unknown at plan time on a rebuild, and the provider can't be configured with it. The new stack reads it from `cloud`'s remote state (output `managed_grafana_endpoint`, same pattern as 0005) and runs after `terraform-cloud` in CI.

**What it manages** (`infra/grafana/dashboards.tf`):

- Folder `Observability`.
- Dashboards "Application Metrics" (uid `application-metrics`, was `service-http`), "Application Logs" (`application-logs`) and "Home" (`home`). JSON lives in `infra/grafana/dashboards/`, exported from AMG without `id` and `version`.
- Org preference: home dashboard = `home`.

The hand-made objects were deleted and recreated by Terraform instead of imported. Import blocks would have had to be removed after the first apply, because on a rebuild there is nothing to import.

Not managed: the datasources. Azure creates both (the AMW integration and `azure-monitor-oob`), and the dashboards pick them through datasource variables, not fixed uids.

**`Grafana Viewer` for the Dev group** (`Hien Test Group Dev`), in `infra/cloud/managed-grafana.tf`. The self-hosted Grafana gives the same group Editor. Here Viewer is enough, because dashboard edits made in the UI would be overwritten anyway.

## How the CI gets a Grafana token

The general mechanism (sign-in vs access token, audiences, token vs role) is in 0027. In short: `azure/login` only gives `az` a session, and the Grafana provider can't turn that into a token. The workflow does it:

1. `az account get-access-token --resource ce34e7e5-…` returns a token with the AMG audience.
2. `::add-mask::` hides it in the logs.
3. `$GITHUB_ENV` carries it to later steps as `GRAFANA_AUTH`, because shell variables don't survive between steps.

## Consequences

- **Change a dashboard in the UI, export its JSON into `infra/grafana/dashboards/`, commit.** Edits made only in the UI are lost on the next apply.
- **`grafana_dashboard` is the legacy resource.** From Grafana 13 the provider suggests the Kubernetes-style dashboard resources. The legacy one still works with the classic JSON model that AMG exports.
- **Running a plan locally** needs your own token in `GRAFANA_AUTH`, the same `az account get-access-token` command. Members of the admin group have `Grafana Admin`.
- **Verified:**
  - A local plan against the live instance first showed only imports and no changes, so the exported JSON matched.
  - After the objects were deleted, CI created all 5 resources. The home page lists both dashboards. The role assignments show Admin for the admin group and CI, and Viewer for Dev.

## References

- https://registry.terraform.io/providers/grafana/grafana/latest/docs
- https://learn.microsoft.com/en-us/azure/managed-grafana/how-to-api-calls
- https://learn.microsoft.com/en-us/azure/managed-grafana/how-to-service-accounts
- https://learn.microsoft.com/en-us/azure/managed-grafana/concept-role-based-access-control
