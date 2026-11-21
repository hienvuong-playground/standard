# 0022 – OTel metrics go to the Azure Monitor workspace as cumulative

## Context

The collector had a second metrics pipeline, `metrics/azure`, that ran `cumulativetodelta` before exporting to Azure Monitor. Its comment said "App Insights needs delta temporality". But these metrics never reach App Insights. The OTLP DCR (`infra/cloud/monitor.tf`) sends the `Custom-Metrics-Otel` stream only to the Azure Monitor workspace (AMW), and Grafana (AMG, 0021) queries it with PromQL. Another project worked without the processor, so the requirement was checked.

## Decision

**Drop `cumulativetodelta`.** A single `metrics` pipeline now exports to both Prometheus remote-write and Azure Monitor (`infra/k8s/gitops/infrastructure/observability/otel/config/otel-collector.yaml`).

Evidence that AMW and Grafana handle cumulative:

- AMG ships a built-in dashboard for it, "Azure / Insights / Applications / OTel / HTTP Cumulative". Its note says to use it "for metrics which are ingested in cumulative temporality", which is "not the default setting for the Application Insights OpenTelemetry distro".
- Microsoft's PromQL guide for OTel metrics in AMW lists the aggregation temporality as "Cumulative or Delta". For cumulative it uses `rate()`/`increase()`.
- The delta requirement in the OTLP ingestion doc is scoped to "Application Insights experiences", i.e. the App Insights portal. The App Insights resource here isn't linked to the AMW, so those views don't read these metrics anyway.

With delta, AMW queries had problems:

- PromQL warnings: "conflicting counter resets during histogram aggregation" and "encountered a mix of histograms and floats".
- `histogram_quantile` dropped points: 15 of 61 minutes for `/livez` in one hour.

The cause wasn't found. One suspect is that `cumulativetodelta` keeps state per collector replica, and there are 2 replicas. Neither problem has shown up since the switch.

Rejected: making the SDK emit delta (`OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=delta`). Prometheus remote-write needs cumulative.

## Consequences

- **Dashboards need cumulative queries** (`rate`/`increase`). The built-in "HTTP Delta" dashboard is wrong for this data. "HTTP Cumulative" is the base of the team dashboard (0023).
- **AMW series changed from delta to cumulative at 2026-11-20 16:28 UTC.** Time ranges that include that moment show wrong numbers until it ages out.
- **App Insights portal metric views would expect delta.** They aren't used.
- **KEDA is unaffected.** It reads the self-hosted Prometheus.
- **Verified 3 minutes after rollout:** AMW matched Prometheus exactly, with no warnings. For `/readyz`: total count 4922 = 4922, 2-minute increase 12 = 12, avg 2.60998 ms, p95 7.75 ms. After a load burst, every route and status code matched as well (0023).

## References

- https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-opentelemetry-best-practices
- https://learn.microsoft.com/en-us/azure/azure-monitor/containers/opentelemetry-protocol-ingestion
- https://opentelemetry.io/docs/specs/otel/metrics/data-model/#temporality
