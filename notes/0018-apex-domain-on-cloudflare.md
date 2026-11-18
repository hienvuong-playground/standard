# 0018 – Apex `hienvuong.com` behind Cloudflare; Cloudflare API tokens

## Context

Before this change, every host lived under `playground.hienvuong.com`. That subdomain was delegated with NS records from the Cloudflare `hienvuong.com` zone to an Azure DNS zone in `rg-playground-manual`. The records there pointed straight at the Gateway PIP, and cert-manager wrote its DNS-01 TXT records there too (0010). Cloudflare served DNS for the parent zone only. Nothing went through its CDN.

0013 worked out what putting Cloudflare in front would take: the records have to move into the Cloudflare zone, and the Universal SSL edge cert covers only the apex and first-level names. That made `www.playground…` and `api.playground…` a problem, and it was left as an open question.

## Decision

### Drop the subdomain, use the apex

All hosts are now first-level: `www`, `api`, `grafana`, `flux` under `hienvuong.com`, plus the apex. Universal SSL covers them all for free, so the 0013 open question is gone. The domain flows from one place: `domain_name` in `infra/cloud/output.tf` comes from the Cloudflare zone, and Flux's `DOMAIN_NAME` drives the cert, the Gateway listeners, the routes, CORS and the Entra redirect URIs. The only hardcoded copy is `VITE_API_URL` in `frontend/.github/workflows/build-push.yml`.

Cloudflare config in Terraform (`infra/cloud/cloudflare.tf`):
- A records for the apex and `*`, pointing at the Gateway PIP. They are `proxied = true` with `ttl = 1` (proxied records must use automatic TTL). Names are written as FQDNs rather than `@`, so a name the API returns in normalized form doesn't show up as drift.
- SSL mode Full (strict). The origin keeps its Let's Encrypt cert (reasons in 0013).
- Caching config: none. The edge honors origin `Cache-Control` (0008).

### cert-manager DNS-01 via Cloudflare

The `ClusterIssuer` uses the `cloudflare` solver (`infra/k8s/gitops/infrastructure/cert-manager/main/issuer.yaml`). The solver only reads a k8s Secret (`apiTokenSecretRef`). It has no way to read the token from a file. So the token goes Key Vault → Secrets Store CSI → Secret:
- `infra/k8s/gitops/infrastructure/cert-manager/secret-provider-class.yaml` syncs the KV secret into Secret `cloudflare-api-token`. It uses the cert-manager managed identity, which already had a federated credential.
- The CSI driver creates the Secret only while some pod mounts the volume. So the cert-manager controller mounts it (`volumes`/`volumeMounts` in `.../cert-manager/init/helm.yaml`). The ServiceAccount gets the `client-id` annotation, as in the AKS docs.
- The SecretProviderClass sits next to `namespace.yaml`, not in `main/`. `cert-manager-init` waits (`wait: true`) for the pod, and the pod can't start without the SecretProviderClass. If the SPC lived in `main/`, which depends on `init`, neither would ever become ready.

### Two Cloudflare API tokens

| Permission (API name) | `infra-ci` (Terraform in CI) | `cert-manager` |
|---|---|---|
| Zone Read | look up the zone by name (`data "cloudflare_zone"`) | the solver finds the zone for `_acme-challenge` by listing zones |
| DNS Write | apex + `*` records | create and delete the TXT challenge |
| Zone Settings Write | `ssl = strict` | – |
| Account API Tokens Write | create the `cert-manager` token | – |
| Zone WAF Write | the `www` challenge rule (below) | – |

The zone permissions apply only to the `hienvuong.com` zone. The dashboard calls "Write" "Edit", and it shows a permission like Zone Read as three dropdowns (Zone / Zone / Read). Permission group IDs come from `GET /accounts/{account_id}/tokens/permission_groups`. They are Cloudflare-wide constants, so they're hardcoded with the name in a comment. A permission group only applies to resources of its own scope. So the account-scoped `Account API Tokens Write` needs its own policy with the account as resource. In one policy with the zone it would do nothing.

`Zone WAF Write` was checked in Cloudflare's OpenAPI schema (`cloudflare/api-schemas`, `x-api-token-group`). The ruleset endpoints accept about 20 permissions, one per product phase, and `Zone WAF Write` is among them. The schema doesn't say which permission goes with which phase. A read-only `GET .../rulesets/phases/http_request_firewall_custom/entrypoint` returned 404 (no ruleset yet) instead of 403, which confirmed the token could access that phase.

**`infra-ci` is created by hand** in the dashboard (Manage Account → Account API Tokens). The first plan had init create it from a bootstrap token that could only manage tokens. Cloudflare refuses that: `1001 sub-token is not allowed to have permissions to manage other tokens`. A token created through the API can never manage tokens. That leaves one hand-made token, and with or without init it would be the same one. Init only copies it into the org secret `CLOUDFLARE_API_TOKEN` (`infra/init/main.tf`, visible to `infra` only), the same way it handles `GH_APP_PEM`. The `terraform-cloud` job passes it as env to plan/apply (`infra/.github/workflows/deploy.yml`).

**`cert-manager` is created by the cloud stack** (`cloudflare_account_token.cert_manager`) and written to Key Vault. The KV secret uses `value`, not `value_wo`. The token is in state anyway via the token resource, and `value` follows a recreated token without bumping a version by hand.

Rejected:
- **Cert-manager token made outside the infra stacks** (by hand or by init), then GitHub secret → ephemeral var → `value_wo` → KV. Infra state stays clean, and the CI token wouldn't need token-management rights. But without init, that means two tokens made by hand, with their permissions documented nowhere but docs, and a rotation that needs a manual version bump. The cert-manager token's rights are a subset of `infra-ci`'s, so putting it in GitHub too wouldn't add exposure. The cost was the manual work.
- **An init-owned Key Vault that CI and cert-manager read at runtime.** That makes the running system depend on init. Init only stands in for manual steps (see CLAUDE.md).
- **Keep the ACME TXT in Azure** via a CNAME from `_acme-challenge` to an Azure zone. No secret in the cluster, but it keeps an Azure zone around just for ACME.

### Bot protection: a Managed Challenge on `www` only

Goal: stop simple bots if it comes cheap. Not a hard requirement.

- **Tried first: Bot Fight Mode**, the Free plan's official bot product (one switch). Rejected:
  - It covers the whole zone and "cannot be bypassed or skipped using WAF custom rules". The docs warn it "may challenge API or mobile app traffic".
  - Verified: a request from an Azure IP (a pod in AKS) got `403 cf-mitigated: challenge` on both `www` and `api`. Excluding a host takes Super Bot Fight Mode plus a Skip rule, which needs Pro.
  - Two API quirks from that attempt:
    - `fight_mode = true` is refused unless `enable_js = true` ("cannot enable Fight_Mode while EnableJS is disabled").
    - Turning `fight_mode` off silently reset `ai_bots_protection` to `disabled`, so Terraform failed with "inconsistent result after apply".
  - `cloudflare_bot_management` can't be destroyed by Terraform. It was dropped from state with a `removed { destroy = false }` block, and that block was deleted once applied. The zone keeps `fight_mode = false` (the default) and `enable_js = true`. `Bot Management Write`, added for this attempt, was removed from the token again.
- **Chosen: a WAF custom rule** (`cloudflare_ruleset.waf_custom` in `infra/cloud/cloudflare.tf`): `http.host eq "www.hienvuong.com"` → `managed_challenge`. Free allows 5 custom rules.
  - Only `www`, because a challenge "fails when the browser expects a non-HTML response, such as an AJAX or XHR (fetch) request". The storefront calls `api` cross-origin, so the `www` clearance cookie isn't sent, and Grafana's own XHR would break when the cookie expires.
  - `/assets/` is not excluded. The frontend is one bundle with no lazy chunks, so every asset loads right after the HTML with the `cf_clearance` cookie. Revisit if code-splitting is added: a chunk loaded after the cookie expires (30 min by default) would get the challenge page.
  - Managed Challenge solves itself for most real browsers. `curl` gets `403 cf-mitigated: challenge` on `www`, including from a home IP.
  - Not a bot detector: it challenges every `www` request and lets Cloudflare decide. Bot score fields (`cf.bot_management.*`) are Enterprise only. Verified bots such as search crawlers are challenged too, and `robots.txt` with them. That's acceptable for a playground with no SEO.
  - Not built from a single guide. The pieces:
    - challenge pages: implement via a WAF custom rule, use Managed Challenge
    - custom rules: Free plan limits
    - the Terraform `cloudflare_ruleset` schema: `phase`, `kind = "zone"`, `action = "managed_challenge"`
    - the `http.host` field reference

## Consequences

- Deliberate playground shortcuts, on top of those in 0015:
  - `infra-ci` can create any token in the account, so it effectively has full Cloudflare account access. It lives in a GitHub org secret, and any workflow on any branch of `infra` can read that. Azure login, by contrast, is pinned to `main`.
  - The `cert-manager` token value is in `cloud.tfstate`. Terraform state should be treated as sensitive regardless.
  - Hardening if wanted: a GitHub Environment restricted to `main` for these secrets. The job's OIDC subject then becomes `environment:<name>`, so init needs another federated credential.
- Rotation: `cert-manager`: replace the token resource and CI updates KV; the CSI pod picks it up on remount or rotation. `infra-ci`: make a new one in the dashboard, update `init/secrets.auto.tfvars`, rerun init.
- Cutover was one switch. The old `playground` names were already dead: the NS delegation had been removed before the switch. The Azure DNS zone and `rg-playground-manual` are deleted by hand. They're outside Terraform, and nothing references them anymore.
- With the challenge rule, `www` can't be hit by scripts: `curl` checks and load tests against `www` get 403. Test origin behaviour with `--resolve` to the PIP, or hit `api` directly.
- Not done yet: lock the origin to Cloudflare, and put Blob images behind Cloudflare (0013).

### How it was verified (2026-11-06)

- Cert `wildcard`: `*.hienvuong.com` + `hienvuong.com`, Ready, issued via Cloudflare.
- `www`: 200. `index.html` returns `no-cache` + `cf-cache-status: DYNAMIC`, so the edge doesn't override it and no Browser Cache TTL change was needed. Hashed assets are `immutable`, `MISS` then `HIT`.
- Apex: 301 to `www`. `http://`: 301 to `https://`. Origin directly (`curl --resolve <host>:443:<PIP>`): 200 from `istio-envoy`.
- API: `no-store`/`DYNAMIC`, and CORS echoes `https://www.hienvuong.com` only. `grafana`/`flux` redirect to Entra with the new redirect URI.

### Checking cert-manager

An issuance creates `Certificate` → `CertificateRequest` → `Order` (ACME) → one `Challenge` per name:

```
kubectl --context=aks-playground -n istio-ingress get certificate,certificaterequest,order,challenge
```

- During a reissue: Certificate `READY=False`, the newest CertificateRequest `False`, Order `pending`, and two Challenges, both for `hienvuong.com`. The wildcard and the apex validate at the same `_acme-challenge` name. The old cert keeps serving from `wildcard-tls` meanwhile.
- When done: Certificate `Ready`, Order `valid`, and the Challenges are deleted. "No resources found" is the success case.
- Wait for it: `kubectl ... -n istio-ingress wait certificate/wildcard --for=condition=Ready --timeout=480s`.
- Names and expiry: `kubectl ... get certificate wildcard -o jsonpath='{.spec.dnsNames} {.status.notAfter}'`.
- Stuck: `kubectl ... describe challenge <name>`. Usually token permissions, or the self-check not yet seeing the TXT.
- Token in place: `kubectl ... -n cert-manager get secret cloudflare-api-token`. It only exists while the cert-manager pod mounts the CSI volume. Issuer: `kubectl ... get clusterissuer letsencrypt` → `READY=True`.

## References

- https://developers.cloudflare.com/fundamentals/api/how-to/create-via-api/ (token creation, permission group scopes)
- https://developers.cloudflare.com/fundamentals/api/reference/permissions/
- https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/limitations/
- https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/
- https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-identity-access
- https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/account_token
- https://developers.cloudflare.com/bots/plans/free/
- https://developers.cloudflare.com/bots/get-started/bot-fight-mode/
- https://developers.cloudflare.com/bots/get-started/super-bot-fight-mode/
- https://developers.cloudflare.com/cloudflare-challenges/challenge-types/challenge-pages/
- https://developers.cloudflare.com/waf/custom-rules/
- https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/ruleset
- https://developers.cloudflare.com/ruleset-engine/rules-language/fields/reference/http.host/
- https://github.com/cloudflare/api-schemas (`x-api-token-group` per endpoint)
