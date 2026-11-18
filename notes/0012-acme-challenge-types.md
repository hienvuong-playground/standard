# 0012 – ACME challenge types and why DNS-01

## Context

Wanted to understand what a "solver" in the cert-manager ClusterIssuer is, what kinds exist, and why wildcard certs force DNS-01. Written down as a memory aid; [0010](0010-cert-manager-dns01-trigger.md) covers when the TXT record gets created.

Config: `infra/k8s/gitops/infrastructure/cert-manager/main/issuer.yaml` (one `dns01` solver, `azureDNS`, managed identity).

## How it works

### Solver = the thing that answers the ACME challenge

Before issuing, Let's Encrypt sends a challenge to prove domain control. The solver performs it (create record/file, wait for validation, clean up). cert-manager supports two:

| | HTTP-01 | DNS-01 |
|---|---|---|
| How | temporary Pod + Ingress serving `http://<domain>/.well-known/acme-challenge/<token>` | TXT `_acme-challenge.<domain>` via DNS provider API |
| Wildcard | no | yes (only option) |
| Needs | domain pointing at cluster, port 80 open to internet | DNS write permission for cert-manager |
| Breaks when | ingress, HTTPS redirect, WAF or firewall blocks the path | DNS propagation is slow, permissions wrong |
| Private cluster | no | yes |

TLS-ALPN-01 exists in the ACME spec but cert-manager doesn't support it.

Several solvers can sit in one issuer; `selector` (`dnsNames` / `dnsZones`) picks which one handles which name.

### Why we use DNS-01

- Need wildcard (`*.playground.hienvuong.com`).
- No dependency on the ingress/Istio gateway being up or reachable.
- Managed identity, so no stored secret.

HTTP-01 stays reasonable for a single public host with no DNS access; not relevant here.

### Why wildcard requires DNS-01

HTTP-01 only proves control of one web server behind one host. A wildcard claims the whole namespace under the domain (incl. subdomains that don't exist yet), and there's no concrete URL to call for `*.example.com`. Writing to the DNS zone is the matching level of control. This is in the CA/Browser Forum Baseline Requirements, so every public CA follows it, not just Let's Encrypt.

### Why a wildcard covers only one label

`*` matches exactly one label, leftmost position only (TLS name matching, RFC 6125). This is not a DNS-01 limit.

| Cert | `app.example.com` | `a.b.example.com` | `example.com` |
|---|---|---|---|
| `*.example.com` | yes | no | no |

So the apex must be listed separately (we do), and deeper levels need their own entry, e.g. `*.dev.example.com`. Invalid forms: `*.*.example.com`, `app.*.example.com`. Each name gets its own `_acme-challenge` TXT.

## In practice

- Least privilege: the cert-manager managed identity should have `DNS Zone Contributor` on the zone only, not on the resource group or subscription.
- If a deeper wildcard is ever needed, add it to `dnsNames` in `infra/k8s/gitops/infrastructure/istio/main/certificate.yaml`; that triggers a re-issue (see 0010).
- Optional hardening not done: CNAME `_acme-challenge` to a dedicated zone and grant cert-manager rights only there. Skipped as overkill for the playground.

## References

- https://cert-manager.io/docs/configuration/acme/
- https://cert-manager.io/docs/configuration/acme/dns01/azuredns/
- https://datatracker.ietf.org/doc/html/rfc8555 (ACME, section 8 challenges)
- https://letsencrypt.org/docs/challenge-types/
