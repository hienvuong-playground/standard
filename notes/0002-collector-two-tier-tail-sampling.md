# 0002 - Two-tier collector for tail sampling

Date: 2026-10-22

## Context

Tail sampling ([0001](0001-trace-tail-sampling.md)) only works when all spans of a trace reach the same collector instance. With more than one replica, spans get split across pods, each pod decides on a partial trace, and traces end up broken. I wanted the collector to be able to run with multiple replicas.

## Decision

Split the collector into two tiers, as described in the official [Scaling the Collector](https://opentelemetry.io/docs/collector/scaling/) guide:

- Tier 1 `otel` (`otel/config/otel-collector.yaml`, 2 replicas): receives everything from the SDK. Metrics and logs are unchanged. Traces go through the `load_balancing` exporter, which hashes by trace ID so a trace always lands on the same tier-2 pod. It finds tier-2 pods via the DNS of the headless service the operator creates.
- Tier 2 `otel-tail-sampler` (`otel/config/otel-tail-sampler.yaml`, 2 replicas): runs `tail_sampling` and exports to Tempo and Azure Monitor.

How a trace ends up on one tier-2 pod:

- The headless service (`clusterIP: None`) makes DNS return every tier-2 pod IP instead of one virtual IP. With a normal Service, kube-proxy/Cilium would pick the pod and tier 1 couldn't control where a trace goes.
- Each tier-1 pod builds a consistent-hash ring from those IPs (CRC32, 200 points per IP) and sends each span to the ring position of `CRC32(traceID)`. All tier-1 pods see the same IPs and use the same hash, so they agree without talking to each other.
- When the tier-2 pod list changes, only ~1/N of traces move. Tier-1 pods refresh DNS independently (every 5s), so for a few seconds they can disagree and split a trace.

Kept simple on purpose: DNS resolver (no RBAC needed) instead of the `k8s` resolver, default queue/retry settings.

Tier 2 has a fixed replica count, no autoscaling: changing it reshuffles in-flight traces between pods (per the load-balancing exporter README).

`tls.insecure: true` on the load-balancing exporter is required: OTLP gRPC clients use TLS by default and tier 2 receives plaintext (same as the Tempo exporter).

## Consequences

- Tier 2 reuses the `otel-collector` ServiceAccount, so the existing workload identity works and Terraform wasn't touched.
- The Tempo NetworkPolicy now only allows tier 2 (`otel-tail-sampler-collector`) to push traces.
- Verified with 100 `/api/hello` + 20 `/api/error`: traces split across both tier-2 pods (48/72), and Tempo and LAW (`OTelSpans`) both show 20/20 errors and 12/100 hello, every trace complete (8 spans).

## References

- https://opentelemetry.io/docs/collector/scaling/
- https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/loadbalancingexporter/README.md
- https://github.com/open-telemetry/opentelemetry-collector/blob/main/config/configtls/README.md
