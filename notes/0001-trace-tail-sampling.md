# 0001 - Trace tail sampling

Date: 2026-10-22

Update: tail sampling moved to a separate collector tier, see [0002](0002-collector-two-tier-tail-sampling.md).

## Context

Every trace was exported to both Tempo and Azure Monitor (App Insights → Log Analytics). Log Analytics bills per GB ingested, and trace volume was driving the cost up.

App Insights is only the trace store Grafana queries (Grafana can't query traces from Log Analytics directly). I don't look at request stats there, so sampling it is fine. Both paths are kept on purpose to try things out side by side.

## Decision

Tail sampling in the OTel collector (`tail_sampling` processor in `infra/.../otel/config/otel-collector.yaml`), applied to the single `traces` pipeline so Tempo and Azure Monitor get the same set:

- Keep every trace that has an `ERROR` span.
- Keep ~10% of the rest (probabilistic).

Why tail and not head sampling: head sampling decides when the trace starts, before it's known whether it fails, so it can't keep errors.

Server-side 4xx are not kept as errors. Per OTel HTTP semconv, a 4xx on a SERVER span is left `UNSET` (the caller's fault), and only 5xx is `ERROR`. A 4xx received by a CLIENT span is `ERROR`. Most 4xx is noise (404, 401), so no extra policy was added.

## Consequences

- Works because there's a single collector pod. Running multiple replicas needs a load-balancing tier so all spans of a trace reach the same instance.
- Span-derived counts (e.g. App Insights request counts) are no longer accurate. KEDA/Prometheus use SDK metrics, so they aren't affected.
- `GET /api/error` was added to the backend to test this. Verified: all error traces kept, ~10% of `/api/hello` kept.

## References

- https://opentelemetry.io/docs/concepts/sampling/
- https://opentelemetry.io/blog/2022/tail-sampling/
- https://opentelemetry.io/docs/specs/semconv/http/http-spans/#status
