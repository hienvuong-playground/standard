# 0030 – Keep health checks out of logs, traces and metrics

## Context

Logs, traces and metrics in AMG were mostly kubelet probes. The backend's readiness probe hits `/readyz` every 10 s and the liveness probe hits `/livez` every 20 s (`backend/deploy/deployment.yaml`). Measured over 24 h on 2026-11-21:

- Logs: 8,437 of 9,561 `OTelLogs` rows (~88%) were probe requests. They were already logged at `debug`, but `LOG_LEVEL=trace` still let them through.
- Traces: 877 of 1,013 sampled traces (~87%) were probes. Each `/readyz` trace has about 10 spans (middleware, `pg-pool.connect`, `pg.query`), and the 10% baseline doesn't look at the route.
- Metrics: 8,156 of about 8,744 requests in AMW (~93%), so every request total, rate and average on the Application Metrics dashboard (0023) was mostly probes. The cost was negligible: the probes make 4 of the 13 `http.server.duration` series, and each series sends one sample per 30 s export whatever the request count. That's about 8 samples/min against 23,779/min for the whole workspace (`EventsPerMinuteIngested`), ~0.03%.

## Where to filter

The OTel blueprint says to process telemetry "as close as possible to the application layer". The Collector is "a fallback measure on applications that could not apply these settings at the SDK level".

- **Logs, in the app.** pino-http's `customLogLevel` returns `silent` for the probe routes (`backend/src/app.js`), and pino-http then skips the "request completed" line. The 5xx and 4xx checks come first, so a failing probe is still logged as `error`/`warn`.
- **Metrics, in the Collector.** Same reason as traces for the hook. SDK Views don't help either: they select instruments by name, type, unit or meter, and `attribute_keys` is "an allow-list of attribute keys", so they can't drop datapoints by attribute value.
- **Traces, in the Collector.** The SDK can't do it while the Operator injects the instrumentation. The injected `autoinstrumentation.js` calls `getNodeAutoInstrumentations()` with no options. `ignoreIncomingRequestHook` is a function, and functions are "in-code only", with no env var or config file. The only other way is a custom auto-instrumentation image. Auto-inject stays, so the Collector is the place.

## Decision

Policies in `infra/k8s/gitops/infrastructure/observability/otel/config/otel-tail-sampler.yaml`. Names and layout follow the KubeCon EU 2026 "observability on budget" example. The drop rule follows OneUptime's "Drop Health-Check Traces Without Hiding Child-Span Errors".

- `keep-errors`: any span with status `ERROR`. Same as before, renamed from `errors`.
- `keep-slow`: trace longer than 2000 ms (first start to last end).
- `drop-health`: `http.route` is `/livez` or `/readyz` AND `not` (any span has status `ERROR`) AND `not` (trace longer than 2000 ms).
- `sample-rest`: 10% of the rest. Same as before, renamed from `baseline`.

How it works:

- A `drop` decision wins over every `sample` decision, whatever the policy order. So failed or slow probes can't be rescued by `keep-errors` or `keep-slow`. They have to be excluded from the drop itself. The `not-slow` threshold matches `keep-slow`, so `drop-health` only drops probes that neither keep policy would keep.
- Sub-policies under `drop` are ANDed. `drop.go` returns Dropped only "if all sub-policies returned a Sampled Decision".
- `not` is allowed under `drop` (`drop_helper.go`). It flips the decision of its one sub-policy (`not.go`).
- `string_attribute` and `status_code` look at every span in the trace. Only the server span carries `http.route`.
- The route key is old HTTP semconv because that's what the backend emits (0023).
- Needs the whole trace in one place. `sampling_strategy` defaults to `trace-complete`, and tier 1 routes by trace ID (0002).

Why slow probes are kept:

- When the backend is idle, probes are the only steady traffic that touches the DB. In 24 h there were about 670 app requests against about 8,430 probe requests.
- If the DB gets slow then, metrics only show that `/readyz` got slower. The trace shows where the time went: `pg-pool.connect` or `pg.query`.
- With real traffic, slow app requests already carry the same pg spans, so slow probes add little.
- It costs almost nothing. In 7 days all 17,460 probes finished within 1 s (`le="1000"` bucket), so normally nothing extra is kept.
- None of the three examples keeps slow probes, but none of them gives a reason either.

Why failed probes are kept: the last 30 days had 40 `/readyz` 5xx traces. Each one carries the cause on its `pg.connect` span, for example `connect ECONNREFUSED …:5432`. Nothing else records it:

- The `/readyz` handler swallows the exception, so the log line only says 503.
- `KubeEvents` (0029) records that the probe failed, but not why. An `Unhealthy` event says `Readiness probe failed: …` with the HTTP result, nothing from the DB.

Metrics: `filter/drop-health` in the `metrics` pipeline of `infra/k8s/gitops/infrastructure/observability/otel/config/otel-collector.yaml`. It drops datapoints whose `http.route` is `/livez` or `/readyz` and whose `http.status_code` is below 400.

- Same cut as logs: failed probes stay, so `/readyz` 503 shows in the failure panels when the DB is down.
- `http.status_code` is an int on the metric (copied from the span attribute, `instrumentation-http` `utils.js`), so `< 400` compares numbers.
- The filter processor can drop datapoints, and "If all datapoints for a metric are dropped, the metric will also be dropped". Its orphaned-telemetry warning is about spans only.
- None of the three examples above covers metrics (the cookbook recipe says "Signals: traces"). No source says where to drop probe metrics, so this is a judgement.

Rejected:

- KubeCon's `drop-health` as written. It matches the route only, so failed probes are dropped too.
- The cookbook's way of keeping failures, which drops only when the HTTP status is 200 (`numeric_attribute`, min = max = 200). It looks only at the server span's HTTP status. A 200 probe with a failed child span would still be dropped. `not` + `status_code` keeps any trace that `keep-errors` would keep.
- The cookbook's extra 1% of successful probes. It adds a sub-policy for data that isn't looked at.
- Excluding probes in the dashboard queries (`"http.route"!~"/livez|/readyz"`). It keeps the data, but it's 36 selectors in `application-metrics.json`, and the built-in AMG dashboards, Explore and any new query would still show probes.
- `filter` processor for traces. It drops single spans and warns that "Dropping a span may lead to orphaned spans if the dropped span is a parent". A probe trace has a parent span with children.

## Consequences

- Successful, fast probes leave no trace and no log line. Failed probes keep both. Slow probes keep the trace.
- A probe that fails with 4xx has span status `UNSET`, so it would be dropped. The probes here only return 200 or 503.
- `keep-slow` keeps every `/api/slow` call (1–3 s delay) that runs over 2 s.
- Metrics lose the latency of successful probes. That was the only metric of DB connect time when there's no real traffic. Slow probes still keep their trace.
- `db.client.operation.duration` still includes the probes' queries (~89% of `select`s in 24 h). It has no `http.route`, so the filter can't tell them apart. No dashboard uses it yet.
- The filter matches old HTTP semconv keys. After a switch to stable semconv it stops matching and probes come back, nothing is lost.
- Checked: the `tail_sampling` block passes `otelcol-contrib validate` on 0.159.0. A bad key in it fails validation, so the check is real.
- Verified after rollout (2026-11-21, new policies live from 16:16 UTC):
  - Logs: once the new backend image was up, the only probe row was the failed one below. The hour before, all 1,078 rows were probes.
  - Successful probes: 0 probe traces in the first minutes, against about one per minute with the old config. Test traffic still flowed: `/api/error` 3 of 3 kept, `/api/hello` sampled.
  - Metrics (filter live at 16:50 UTC): the `/readyz` 200 and `/livez` 200 series stopped at 16:50:20. Other routes, `/readyz` 503 and `db.client.operation.duration` kept updating. The full collector config passes `validate`, and a local run on 0.159.0 dropped only the successful probe datapoints.
  - Postgres restart (`az postgres flexible-server restart`, 16:21:30–16:22:35) produced both kept cases:
    - Failed probe, 16:21:52: `GET /readyz` 503 with status `Error`. Its `pg.connect` span says `the database system is shutting down`. The log row is `error`, but it only says "failed with status code 503".
    - Slow probe, 16:22:02: `GET /readyz` 200, no error, 2,274 ms. `pg.connect` took 2,264 ms while the DB came back, and the query 9 ms. Only the trace shows that.

## References

- https://opentelemetry.io/docs/guidance/blueprints/managed-telemetry-platforms-for-k8s-workloads/#guideline-4
- https://opentelemetry.io/docs/guidance/blueprints/managed-telemetry-platforms-for-k8s-workloads/#action-4
- https://github.com/pinojs/pino-http/blob/master/README.md (`customLogLevel`, `silent`)
- https://github.com/open-telemetry/opentelemetry-js/blob/main/experimental/packages/opentelemetry-instrumentation-http/README.md#declarative-configuration
- https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/#using-customized-or-vendor-instrumentation
- https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.159.0/processor/tailsamplingprocessor/README.md#policy-decision-flow
- https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.159.0/processor/tailsamplingprocessor/internal/sampling/drop.go
- https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.159.0/processor/tailsamplingprocessor/drop_helper.go
- https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.159.0/processor/tailsamplingprocessor/internal/sampling/not.go
- https://oneuptime.com/blog/post/2026-08-25-drop-health-check-traces-safely/view
- https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.159.0/processor/filterprocessor/README.md#warnings
- https://opentelemetry.io/docs/specs/otel/metrics/sdk/#view
- https://azure.microsoft.com/pricing/details/monitor/
- https://github.com/pavolloffay/kubecon-eu-2026-opentelemetry-observability-on-budget/blob/main/06-tail-based-sampling.md
- https://github.com/jpkrohling/otelcol-cookbook/tree/main/starters/remove-health-checks
