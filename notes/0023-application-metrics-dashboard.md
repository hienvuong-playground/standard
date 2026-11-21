# 0023 – Application Metrics dashboard, from the built-in HTTP Cumulative

## Context

AMG (0021) ships "Azure / Insights / Applications / OTel / HTTP Cumulative". I compared it with the self-hosted Prometheus, which receives the same metrics through remote-write. Most formulas are right, but several panels show wrong numbers. Built-in dashboards are provisioned by Azure, so they can't be saved in place and Azure overwrites them. The fixes therefore live in a copy:

- Title "Application Metrics", uid `application-metrics`, folder Observability (0025).
- First made through the Grafana HTTP API, now in Terraform: `infra/grafana/dashboards/application-metrics.json` (0026).

## Why the numbers were off

The backend's Node auto-instrumentation emits the old HTTP semconv: `http.server.duration` in ms, with labels `http.method` and `http.status_code`. The dashboard is written for the stable semconv (`http.server.request.duration`, in seconds). It has an `or` fallback to the old metric, but that branch isn't converted to seconds. The setup isn't wrong, and no SDK setting fixes every panel (see "Rejected" below).

## Changes

- **Latency was 1000× off.**
  - Fix: divide the old-metric branch by 1000 and set unit `s` on every response-time panel (avg and p95, server and client).
  - Renamed the table title from "Average Server Response Time (milliseconds)" to "(seconds)".
  - The client panels have no data, because the backend makes no outbound HTTP calls.
- **Bar gauges "Average Server Response Time" and "… by Route".**
  - Average: Grafana's `mean` of per-window averages is unweighted, so a minute with 1 request counted as much as a minute with 99. `/api/storefront` showed 7.39 ms; the real value was 8.58 ms. Now an instant query `histogram_avg(increase(…[$__range]))` with reduce `lastNotNull`.
  - Scale: set `min = 0`. The automatic min was the smallest value, so the shortest bar was always empty.
  - Threshold: red at `0.5`. The built-in `500` only meant 500 ms while values were in ms.
- **"Server Requests over time".**
  - Problem: `increase(…[$__rate_interval])` evaluated every 30 s gives 2-minute windows, so each request was counted in about 4 points.
  - Fix: `increase(…[$__interval])` with the panel's min interval set to `1m`. Metrics are exported every 30 s, and `increase` needs 2 samples per window.
  - Where the numbers come from: `$__rate_interval` = max(`$__interval` + scrape interval, 4 × scrape interval). The datasource scrape interval is 30 s, set by Azure.
- **Service variable** now defaults to All. It was empty.

Left as is:

- A regex typo `"5..$.."` in the stable-semconv branch of the failure panels, and old label names in the group-bys. Both only matter after a semconv switch.
- Totals show decimals, because `increase` extrapolates.

Rejected: `OTEL_SEMCONV_STABILITY_OPT_IN=http`.

- It fixes the units, but the stable-metric branch still groups and filters by the old label names. The method and status panels would lose their grouping, and the failure panels would go empty.
- It also renames the Prometheus metric, which breaks the KEDA query in `backend/deploy/backend-scaling.yaml`.
- `http/dup` keeps KEDA working but doesn't fix the dashboard. After `sum by`, both branches have the same labels, so `or` keeps the old ms branch.

## Consequences

- **Verified against Prometheus** after a curl burst that covered 200/400/404/503 and POST:
  - Totals by route, status and method, rps, avg and p95 by route all match. Small differences come from `increase` extrapolating at a series start.
  - The bar gauges match `sum(increase(_sum)) / sum(increase(_count))` per route.
- **Treat built-in dashboards as templates.** Check their numbers against a reference before trusting them.

## References

- https://opentelemetry.io/docs/specs/semconv/http/http-metrics/
- https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/#use-__rate_interval
- https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/bar-gauge/
