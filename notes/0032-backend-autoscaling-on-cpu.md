# 0032 – Scale the backend on CPU only

## Context

The first plan was to scale the backend on request count with KEDA. On second thought, request count doesn't show whether a pod is out of capacity. Many requests don't mean the app is slow, and a slow app doesn't mean the app is the problem. The need is plain: add pods when the existing ones can't keep up. That's a common problem, so the question was what the documented default is for a plain HTTP app.

## Decision

Scale on one metric, CPU utilization at 60%, 1 to 5 pods. No request count, no latency, no memory. Config: `backend/deploy/backend-scaling.yaml`.

It's a KEDA ScaledObject with a single `cpu` trigger, not a hand-written HPA. KEDA creates the HPA (`keda-hpa-backend`) from it, so the scaling is the same. The AKS KEDA add-on is already on (`infra/cloud/aks.tf`), and event-based triggers are planned, so all scaling stays in one kind of object. For a `cpu` trigger, the HPA reads CPU from metrics-server, not through KEDA, so CPU scaling keeps working if the KEDA operator is down.

Why not latency: a slow response can come from the DB. Scaling the app then opens more connections and sends more queries to a DB that is already slow. CPU avoids this on its own. While pods wait on a slow DB, they are idle on I/O, so CPU stays low and HPA doesn't scale.

Why not request count: requests don't cost the same. A health check and a query joining 5 tables both count as 1, and the mix changes over time. The SRE book says QPS "often makes for a poor metric", because "Even if these metrics perform adequately at one point in time, the ratios can change". Its answer is the same as this decision: "we've found that simply using CPU consumption as the signal for provisioning works well". Request-based scaling isn't wrong in principle. Cloud Run scales on "the average CPU utilization and the average request concurrency". It needs a known per-pod capacity from a load test, though, and we don't need that yet.

Why not memory:

- On a GC runtime, CPU already covers it. The SRE book: "In platforms with garbage collection, memory pressure naturally translates into increased CPU consumption."
- With several metrics, HPA picks the highest replica count ("The largest scale is selected for the autoscale action"). V8 doesn't give heap back right after load drops, so a memory metric would keep replicas up after a spike. It would scale up but not back down. This is our own reasoning, not from a doc.
- Memory that keeps growing is usually a leak, and more pods would only hide it.
- Cloud Run doesn't offer it at all: "Cloud Run doesn't support scaling based on memory utilization."

## Consequences

- `resources.requests.cpu` must be set. Utilization is "a percentage of that Pod's CPU or memory requests", and without requests HPA "takes no action related to that metric". A wrong request makes the percentage meaningless.
- CPU lags, so the target needs headroom for the time new pods take to start. GKE: "consider the buffer size to be enough for handling requests for two or three minutes during a spike." Cloud Run's default is 60%. Around 60–70% is a sane start.
- Node runs JS on one thread, so a pod hardly uses more than 1 core. A request above 1 core means utilization never reaches the target and HPA never scales. Keep the request at 1 core or less.
- The request is still 50m (`backend/deploy/deployment.yaml`), so 60% means about 30m per pod, around 3% of what one Node pod can do. Pods scale out long before they are busy. Kept on purpose until a load test shows the CPU at which latency starts to rise. Run that test with one pod, or HPA hides the per-pod limit by scaling out early.
- No other HPA may target `backend`, and the Deployment must not set `replicas`, or Flux and the HPA fight over the count.
- Blind spot: a pod can be saturated while CPU stays low, for example when its `pg` pool is exhausted and requests queue for a connection. CPU HPA won't react. The fix there is the pool size or the DB, not more pods. It's not a reason to drop CPU, only something to check when the app is slow and HPA does nothing.
- Scale-down waits 5 minutes by default ("defaults to 300 seconds (five minutes) in GKE"), so replicas stay up for a while after a spike.

## References

- Google SRE book, Handling Overload, "The Pitfalls of 'Queries per Second'": https://sre.google/sre-book/handling-overload/
- GKE, Horizontal Pod autoscaling (per-Pod resources, multiple metrics, thrashing): https://docs.cloud.google.com/kubernetes-engine/docs/concepts/horizontalpodautoscaler
- GKE, cost-effective Kubernetes applications, Horizontal Pod Autoscaler (target buffer): https://docs.cloud.google.com/architecture/best-practices-for-running-cost-effective-kubernetes-applications-on-gke
- Cloud Run, instance autoscaling: https://docs.cloud.google.com/run/docs/about-instance-autoscaling
- Kubernetes, Horizontal Pod Autoscaling: https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/
- KEDA, CPU scaler: https://keda.sh/docs/2.x/scalers/cpu/
