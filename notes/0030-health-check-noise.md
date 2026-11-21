# 0030 – Keep health checks out of logs and traces

## Context

Logs, traces and metrics in AMG were mostly kubelet probes. The backend's readiness probe hits `/readyz` every 10 s and the liveness probe hits `/livez` every 20 s (`backend/deploy/deployment.yaml`). Measured over 24 h on 2026-11-21:

- Logs: 8,437 of 9,561 `OTelLogs` rows (~88%) were probe requests. They were already logged at `debug`, but `LOG_LEVEL=trace` still let them through.
- Traces: 877 of 1,013 sampled traces (~87%) were probes. Each `/readyz` trace has about 10 spans (middleware, `pg-pool.connect`, `pg.query`), and the 10% baseline doesn't look at the route.
- Metrics: about 8,430 of 9,100 requests in Prometheus.

## Where to filter

The OTel blueprint says to process telemetry "as close as possible to the application layer". The Collector is "a fallback measure on applications that could not apply these settings at the SDK level".

- **Logs, in the app.** pino-http's `customLogLevel` returns `silent` for the probe routes (`backend/src/app.js`), and pino-http then skips the "request completed" line. The 5xx and 4xx checks come first, so a failing probe is still logged as `error`/`warn`.
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

Rejected:

- KubeCon's `drop-health` as written. It matches the route only, so failed probes are dropped too.
- The cookbook's way of keeping failures, which drops only when the HTTP status is 200 (`numeric_attribute`, min = max = 200). It looks only at the server span's HTTP status. A 200 probe with a failed child span would still be dropped. `not` + `status_code` keeps any trace that `keep-errors` would keep.
- The cookbook's extra 1% of successful probes. It adds a sub-policy for data that isn't looked at.
- `filter` processor. It drops single spans and warns that "Dropping a span may lead to orphaned spans if the dropped span is a parent". A probe trace has a parent span with children.

## Consequences

- Successful, fast probes leave no trace and no log line. Failed probes keep both. Slow probes keep the trace.
- A probe that fails with 4xx has span status `UNSET`, so it would be dropped. The probes here only return 200 or 503.
- `keep-slow` keeps every `/api/slow` call (1–3 s delay) that runs over 2 s.
- Metrics aren't filtered yet. The probe routes still show up in the Application Metrics dashboard (0023).
- Checked: the `tail_sampling` block passes `otelcol-contrib validate` on 0.159.0. A bad key in it fails validation, so the check is real.
- After rollout, expect `GET /livez` / `GET /readyz` spans in `OTelSpans` only for failed or slow probes, and no new 2xx probe rows in `OTelLogs`.

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
- https://github.com/pavolloffay/kubecon-eu-2026-opentelemetry-observability-on-budget/blob/main/06-tail-based-sampling.md
- https://github.com/jpkrohling/otelcol-cookbook/tree/main/starters/remove-health-checks
