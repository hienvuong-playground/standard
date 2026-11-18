# 0013 – Cloudflare as CDN: DNS has to move to Cloudflare; two TLS legs

## Context

The goal is to put Cloudflare in front of the Istio Gateway PIP as a CDN. Nothing is implemented yet. This records why the setup has to look a certain way, so it isn't re-derived later.

Current state (checked with `dig`, 2026-11-02):
- `hienvuong.com` is registered at Cloudflare and uses Cloudflare nameservers.
- `playground.hienvuong.com` is delegated to Azure DNS via NS records in the Cloudflare zone. The Azure zone is in `rg-playground-manual`.
- The origin is the Gateway PIP (`infra/cloud/gateway.tf`). The wildcard + apex cert comes from Let's Encrypt via cert-manager DNS-01 on Azure DNS (`infra/k8s/gitops/infrastructure/cert-manager/main/issuer.yaml`, `infra/k8s/gitops/infrastructure/istio/main/certificate.yaml`, see 0010).

## Decision

### DNS: the records must live in the Cloudflare zone

- When Cloudflare is the authoritative DNS, one proxied record covers both legs. DNS answers return Cloudflare anycast IPs (client → edge), and the record's real value tells the edge where the origin is (edge → PIP). Turning on "Proxied" is all it takes.
- With the records kept elsewhere (Azure DNS), you need two pieces: a CNAME at Azure → `<host>.cdn.cloudflare.net`, plus an origin record on the Cloudflare side. That is "partial (CNAME) setup", and it's Business/Enterprise only.
- Cloudflare also has to know that you own the hostname. Anycast IPs are shared across all customers, so the edge picks config by Host/SNI, and it has to issue an edge cert for that name. As the authoritative DNS, it can verify both itself. Front Door works the same way (CNAME to `*.azurefd.net` + TXT validation), except that Azure doesn't require moving DNS.
- A subdomain delegated outside Cloudflare gets no CDN: "Cloudflare's CDN and security services are not applied to delegated subdomains". A separate child zone for `playground` (subdomain setup) is Enterprise only.
- If the domain were registered at Namecheap instead: full setup, which means pointing the registrar's nameservers to Cloudflare. It's the only option on Free/Pro. The registrar doesn't matter otherwise.

**Plan**: move the `playground` records into the Cloudflare `hienvuong.com` zone (Terraform `cloudflare` provider), remove the NS delegation, drop the Azure zone, and switch the cert-manager DNS-01 solver to Cloudflare (API token with `Zone:DNS:Edit` + `Zone:Zone:Read`).

Caching needs nothing new: Cloudflare honors origin `Cache-Control` (0008). The only rule is not to add Edge Cache TTL rules.

### TLS: two separate legs, each with its own cert

```
Browser ──TLS #1──> Cloudflare edge ──TLS #2──> Istio Gateway (PIP)
         edge cert                     origin cert (wildcard-tls)
```

What a cert is for: it proves "this is really host X". Encryption alone doesn't help if the other side could be anyone (a MITM on wifi or at the ISP).

**Leg 1 (browser → edge)** has to be trusted by every browser, so it needs a public-CA cert. Cloudflare's free Universal SSL issues one (from Let's Encrypt, Google Trust Services or SSL.com). Without a valid cert: `NET::ERR_CERT_COMMON_NAME_INVALID`.

- **Universal SSL covers only the apex and first-level subdomains**: "`www.sub.example.com` are not covered and will not serve a valid certificate."
  - `playground.hienvuong.com`: covered.
  - `www.playground.hienvuong.com` and `api.playground.hienvuong.com`: **not covered**.
- The root cause is TLS, not Cloudflare: a wildcard matches exactly one label, and `*.*` doesn't exist. Our own cert has the same limit: `*.playground.hienvuong.com` wouldn't cover `x.api.playground.hienvuong.com`.
- Options for deeper names:
  - Advanced Certificate Manager (paid add-on) + Total TLS, which issues a cert per proxied hostname.
  - Upload a custom cert: Business/Enterprise only.
  - Rename hosts to the first level.
  - Keep `www`/`api` DNS-only, so browsers talk to the Gateway directly with `wildcard-tls`.
- Not yet decided. It affects the "www is canonical" choice in 0011.

**Leg 2 (edge → origin)** only has to be trusted by Cloudflare. The SSL mode decides what's accepted:

| Mode | Edge → origin | Without a valid origin cert |
|---|---|---|
| Flexible | plain HTTP | Here: redirect loop (`ERR_TOO_MANY_REDIRECTS`), because `http-redirect.yaml` sends HTTP → HTTPS. The edge → Azure leg is also plaintext. |
| Full | HTTPS, cert not checked | Self-signed is accepted. Encrypted, but the origin can be impersonated. |
| Full (strict) | HTTPS, cert checked | 525 (handshake failed) or 526 (invalid cert) |

- **Use Full (strict).** The existing Let's Encrypt cert qualifies.
- Strict accepts a public-CA cert or a **Cloudflare Origin CA** cert. That one isn't self-signed: it's signed by a private CA that only Cloudflare trusts.
- **Keep Let's Encrypt, not Origin CA.** With Origin CA, "site visitors may see untrusted certificate errors if you pause Cloudflare or disable proxying". That would break any host kept DNS-only (see leg 1). Origin CA also sends no expiry notices, while cert-manager renews by itself.

## Consequences

- The DNS move and the solver switch risk DNS/cert downtime during cutover. Do it well inside the cert's validity (current one expires 2026-12-10, see 0010).
- Order: Cloudflare records as DNS-only → switch the solver and test a renewal → remove the delegation and the Azure zone → turn on Proxied + Full (strict) → `curl -I` and check `cf-cache-status` and `Cache-Control`.
- Later: lock the origin to Cloudflare (allowlist Cloudflare IPs, Authenticated Origin Pulls, or Tunnel). Put Blob images behind Cloudflare via a custom domain + origin rule for Host/SNI, since the Blob cert is `*.blob.core.windows.net`.
- Open: how to handle the second-level hostnames (leg 1 options above). Resolved in 0018: the subdomain was dropped and every host is first-level under the apex.

## References

- https://developers.cloudflare.com/dns/manage-dns-records/reference/proxied-dns-records/
- https://developers.cloudflare.com/dns/manage-dns-records/how-to/subdomains-outside-cloudflare/
- https://developers.cloudflare.com/dns/zone-setups/full-setup/
- https://developers.cloudflare.com/dns/zone-setups/partial-setup/
- https://developers.cloudflare.com/dns/zone-setups/subdomain-setup/
- https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/limitations/
- https://developers.cloudflare.com/ssl/edge-certificates/additional-options/total-tls/
- https://developers.cloudflare.com/ssl/edge-certificates/custom-certificates/
- https://developers.cloudflare.com/ssl/reference/certificate-authorities/
- https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/
- https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/
- https://developers.cloudflare.com/ssl/troubleshooting/too-many-redirects/
- https://developers.cloudflare.com/fundamentals/security/protect-your-origin-server/
- https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/
