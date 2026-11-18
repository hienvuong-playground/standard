# 0011 – Frontend moved from Static Web App to AKS; www canonical, apex redirects

## Context

The frontend ran on Azure Static Web App (SWA), the backend on AKS behind the Istio Gateway. The aim was one deploy model for both (image in ACR, SHA pinned by CI, Flux), more control and observability at the origin, and to also serve the apex `playground.hienvuong.com`, not only `www`.

## Decision

### Hosting

- nginx (`nginxinc/nginx-unprivileged`) serving the Vite `dist/`: `frontend/Containerfile`, `frontend/nginx.conf`. nginx.conf is the translation of `staticwebapp.config.json` described in 0008.
- `VITE_API_URL` is a build arg (`frontend/.github/workflows/build-push.yml`). Vite inlines it, so an image is tied to one API origin.
- CI and deploy are copies of the backend's: `build-push.yml` builds, pushes and pins the tag in `frontend/deploy/kustomization.yaml` (0004). Flux config: `infra/k8s/terraform/flux.tf` (`frontend`). RBAC for Flux: `infra/k8s/gitops/teams/frontend/`.

### Hostnames

- **www is canonical, the apex 301s to www.** It keeps the existing URL and CORS origin, so cutover needed no CORS change. The apex is shorter, but either works for a playground.
- Apex DNS: alias A record `@` → gateway PIP (`infra/cloud/gateway.tf`). A CNAME isn't allowed at the apex. `www` has no record of its own; it falls under `*`.
- The cert already covered the apex (`dnsNames` in `certificate.yaml`), so no reissue was needed (see 0010).

### Gateway listeners: explicit apex listeners

`*.${DOMAIN_NAME}` doesn't match the apex in Gateway API, so new listeners `http-apex` / `https-apex` were added (`infra/k8s/gitops/infrastructure/istio/main/gateway.yaml`).

Rejected: drop `hostname` from the existing listeners (matches every host). It works and is shorter, but:
- **Lost guardrail.** `https` allows routes from all namespaces. With a hostname, a route can only claim names under the domain. Without one, any namespace's route can claim any hostname, and a route that forgets `hostnames` catches every host, including the bare IP.
- **Cert served for any SNI.** Scanners or foreign domains pointed at the IP get the wildcard cert and a 404 instead of a failed handshake. The names are public in CT logs anyway, so this is minor.
- **HTTP redirect catches any Host**, including `http://<IP>`. Harmless.

Small harm in a single-user cluster; explicit was chosen for clarity and to keep the guardrail.

### Redirects: split between infra and frontend

The apex chain is `http://apex → https://apex → https://www`, two hops. HTTPS on the same host comes first, which is the order HSTS preload expects.

- `infra/k8s/gitops/infrastructure/istio/main/http-redirect.yaml`: HTTP → HTTPS for every host, attached to `http` and `http-apex`.
  - A platform rule, owned by infra.
  - It must live in `istio-ingress`: the port 80 listeners use `allowedRoutes: from: Same`, so a route from another namespace would be rejected.
- `frontend/deploy/apex-redirect.yaml`: apex → www, attached to `https-apex` (`from: All`).
  - The frontend's own choice of canonical host, owned by the frontend repo, the same way the backend owns `api.` (`backend/deploy/httproute.yaml`).
  - Rejected: putting it in infra next to `http-redirect.yaml`. Infra would then have to know the frontend's canonical host.

## Consequences

- **Asset mismatch during rolling updates (accepted).** SWA switched deployments atomically; a rolling update doesn't. While old and new pods both run, a browser can get the new `index.html` and then request `/assets/<new-hash>.js` from an old pod, which returns 404. The window is seconds per deploy, and a reload fixes it. Fixes if it ever matters:
  - ship the previous build's assets in the image, or
  - serve `/assets` from Blob (like 0006).
- The frontend is now down whenever the cluster or the Gateway is down (e.g. AKS stopped). SWA was independent of it.
- **Cutover:**
  - The manual `www` CNAME → SWA in the zone (`rg-playground-manual`, not in Terraform) was deleted, so `www` falls back to `*`.
  - The SWA custom domain was removed by hand.
  - Then removed: `static-web-app.tf`, the `STATIC_WEB_APP_HOST` substitution, the SWA origin in backend CORS, `frontend/.github/workflows/deploy.yaml` and `staticwebapp.config.json`.
- **First-apply race.** `teams/frontend/iam.yaml` (in the `cluster` kustomization) needs the `frontend` namespace, which the Terraform Flux config creates. Flux applied first and failed with "namespaces frontend not found". It recovered on its own after the 10-minute `retryInterval`. This is expected for any new team namespace.
- **Verified with curl, before DNS cutover via `--resolve`:**
  - `/` and SPA routes: 200 with `no-cache`
  - hashed assets: `immutable`
  - missing asset: 404 without `Cache-Control`
  - apex: 301 to www with path and query kept
  - `http://` apex: 301 to https
  - API CORS echoes `https://www...`

## References

- https://gateway-api.sigs.k8s.io/reference/spec/#listener (hostname, wildcard matching)
- https://gateway-api.sigs.k8s.io/guides/http-redirect-rewrite/
- https://learn.microsoft.com/en-us/azure/dns/dns-alias (apex alias record)
- https://hub.docker.com/r/nginxinc/nginx-unprivileged
