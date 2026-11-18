# 0010 – When cert-manager creates the DNS-01 TXT record

## Context

The wildcard cert (`*.playground.hienvuong.com` + apex) is issued by Let's Encrypt through cert-manager using DNS-01: cert-manager writes a TXT record into the Azure DNS zone. This was set up earlier, but it was forgotten when that record actually gets created. Written down so it's not re-derived next time.

Update (0018): the domain is now `hienvuong.com` and the TXT goes to the Cloudflare zone via the `cloudflare` solver. Names and the zone below are from before; the trigger logic is unchanged.

Config:
- Certificate: `infra/k8s/gitops/infrastructure/istio/main/certificate.yaml`
- ClusterIssuer (ACME, `profile: tlsserver`, Azure DNS solver with managed identity `MI_CERT_MANAGER`, zone in `rg-playground-manual`): `infra/k8s/gitops/infrastructure/cert-manager/main/issuer.yaml`

## How it works

### The TXT record is created only when the cert is (re)issued

Not on a schedule, not on every Flux sync. Re-issuance happens when:

1. **Renewal time.** The normal case. With no `renewBefore` set, cert-manager renews "2/3 through the X.509 certificate's duration" and records it in `status.renewalTime`.
2. **Certificate spec changes** to one of: `commonName`, `dnsNames`, `ipAddresses`, `uris`, `emailAddresses`, `subject`, `isCA`, `usages`, `duration`, `issuerRef`.
3. **The Secret (`wildcard-tls`) is missing or broken**: first creation, deleted secret, rebuilt cluster.
4. **Manual**: `cmctl renew wildcard -n istio-ingress`.

`rotationPolicy: Always` doesn't trigger anything. It only means each issuance gets a new private key.

### Flow once triggered

```
Certificate → CertificateRequest → Order (ACME) → Challenge (one per name)
  → TXT _acme-challenge.playground.hienvuong.com written to the Azure DNS zone
  → cert-manager self-check waits for propagation → LE validates
  → cert written to Secret wildcard-tls → TXT deleted
```

- The wildcard and the apex validate at the same name, `_acme-challenge.playground.hienvuong.com`, so that record briefly holds two TXT values.
- Only the latest CertificateRequest/Order stay in the cluster. Challenges are deleted after they succeed, so normally there's nothing to see.

### Cert lifetime

`profile: tlsserver` gives **45-day** certs (`classic` is 90). Observed on 2026-11-01:

| | |
|---|---|
| notBefore | 2026-10-26 |
| notAfter | 2026-12-10 |
| renewalTime | 2026-11-25 (2/3 of 45 d ≈ day 30, 15 days before expiry) |

So the TXT record appears about every 30 days.

## In practice

- Adding a listener or route that uses the existing `wildcard-tls` doesn't touch DNS. Only edits to `certificate.yaml` (e.g. a new name in `dnsNames`) cause a new issuance.
- If renewal fails, there's a 15-day buffer before expiry. Places to check:
  - `kubectl --context aks-playground -n istio-ingress get certificate,certificaterequest,order,challenge`
  - a stuck Challenge's events (usually DNS permissions of the cert-manager identity, or propagation)
- Check current state:
  - `kubectl --context aks-playground -n istio-ingress get certificate wildcard -o jsonpath='{.status.notAfter} {.status.renewalTime}'`

## References

- https://cert-manager.io/docs/usage/certificate/ (renewal, reissuance triggers, rotationPolicy)
- https://cert-manager.io/docs/configuration/acme/dns01/azuredns/
- https://letsencrypt.org/docs/profiles/ (tlsserver = 45 days)
