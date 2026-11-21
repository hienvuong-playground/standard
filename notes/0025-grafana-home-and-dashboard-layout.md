# 0025 – AMG home page and dashboard layout

## Context

When team members open AMG (0021), they should land on the team dashboards (0023, 0024), not on Grafana's welcome page or the list of built-in dashboards. Infrastructure dashboards will follow.

## Decision

- **Folder `Observability`** (uid `observability`) holds the team dashboards. The built-in ones stay in `Azure Monitor`.
- **Tags:**
  - `playground` goes on every team dashboard. The top-bar link "Playground dashboards" lists them.
  - Each dashboard also gets a layer tag, `application` or `infrastructure`.
  - `otel` was tried first and dropped, because the built-in "Azure Functions Health" carries it too.
- **Names:** "Application Metrics", "Application Logs", and later "Infrastructure …".
- **Dashboard `Home`** (uid `home`, General folder):
  - Two dashboard-list panels: "Application" (tag `application`) and "Infrastructure" (tag `infrastructure`, the Kubernetes dashboards from 0028).
  - Tagging a new dashboard is enough for it to appear.
  - Set as the org home in Administration → General → Default preferences → Home Dashboard (API: `PATCH /api/org/preferences`).

Rejected:

- Stars, because they are per user.
- One combined dashboard, because metrics and logs use different datasources and variable sets.

## Consequences

- **Preferences are resolved user > team > org.** Anyone who sets a personal home won't see this page.
- **In a dashboard-list panel, `folderUID: ""` means "General folder only", not "any folder".** The lists stayed empty until the key was removed.
- **Org preferences have no history.** Dashboards do (Settings → Versions). Before this change, no home dashboard was set, so Grafana showed its built-in welcome page.
- **Team members need `Grafana Viewer` on AMG.** The Dev group has it (0026).
- **Everything here was first made by hand through the Grafana API.** The folder, the three dashboards and the org preference are now in Terraform (`infra/grafana/`, 0026).

## References

- https://grafana.com/docs/grafana/latest/administration/organization-preferences/
- https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/dashboard-list/
