# 0016 – Rebuild from scratch: ordered steps and `bootstrap.sh`

## Context

The goal of 0015 is to delete the whole environment and bring it back without having to remember anything. Each note covers its own piece, but nothing listed the order, the manual steps in between, or the rights checked up front. A rebuild that fails halfway on a missing right or an unregistered resource provider is exactly what this should prevent.

## `bootstrap.sh`

Everything needed is either in code or in the steps below. Anything not listed is in code.

**`infra/init/bootstrap.sh`** does what has to happen before init can run. It is idempotent and fails before creating anything if a right is missing:
1. **Preflight**, reporting every problem at once: `Owner` on the subscription; `Global Administrator` in Entra (via Graph `transitiveMemberOf`, so a role through a group counts; a PIM-eligible role must be activated first); `init/secrets.auto.tfvars` has `github_token`, `github_app_pem` and `teams_alerts_webhook_url`; the PAT has scopes `admin:org` and `repo`; the PAT owner is admin of the org; the Cloudflare token is active, can list permission groups (Account API Tokens) and can read the `hienvuong.com` zone (DNS and Zone Settings Edit can't be checked without writing). Also prints the subscription, so a wrong `az login` is visible.
2. **Registers the Azure resource providers.** azurerm 5.x registers none by default (`resource_provider_registrations = "none"`), and a new subscription has only a few registered, so the first apply would fail halfway. The list is derived from the resource types in `infra/init`, `infra/cloud`, `infra/k8s/terraform`; Flux on AKS needs `Microsoft.Kubernetes` + `Microsoft.KubernetesConfiguration`. Adding a resource of a new kind = add its namespace to the script.
3. **Creates the storage account for init's own state** (why it's outside Terraform: 0015) in `rg-playground-init-state`, names read from `infra/terraform-backend/init.tfbackend`. Shared keys off, blob versioning on. Grants the runner `Storage Blob Data Contributor` on it (`Owner` has no blob data-plane rights; the backend uses Entra auth) and retries until the role has propagated.

Rejected for the script: the GitHub Actions PR setting (Terraform has resources for it, so it lives in init instead).

## Steps, in order

1. **GitHub org and repos.** Org `hienvuong-playground` with repos `infra`, `frontend`, `backend` (code pushed). Their numeric IDs go into the OIDC subjects, so if the org or a repo was recreated, update `github_org` / `github_repo_ids` in `infra/init/locals.tf` (lookup commands in the comment there). The org name is also in `infra/k8s/terraform/locals.tf`.
2. **Domain and Cloudflare token (manual).** `hienvuong.com` is registered at Cloudflare and uses its nameservers; the zone itself is outside Terraform, everything in it (records, SSL mode, the cert-manager token) is code (`infra/cloud/cloudflare.tf`). Create the account-owned token `infra-ci` in the dashboard (Manage Account → Account API Tokens) with two policies: Account API Tokens Edit on the account, and Zone Read + DNS Edit + Zone Settings Edit + Zone WAF Edit on the `hienvuong.com` zone (why each, and why it can't be created from code: 0018). If the account or zone was recreated, update `CLOUDFLARE_ACCOUNT_ID` in `infra/init/bootstrap.sh`.
3. **GitHub manual pieces.**
   - Classic PAT with `admin:org` and `repo`.
   - GitHub App + PEM (permissions and installation in 0003). Put its App ID and installation ID in `infra/k8s/terraform/providers.tf`.
4. **`infra/init/secrets.auto.tfvars`** with `github_token`, `github_app_pem`, `cloudflare_api_token` and `teams_alerts_webhook_url`. The Teams URL comes from a Workflows webhook made by hand in the alerts channel ("Send webhook alerts to a channel", add a co-owner; 0031). It survives a rebuild, since Teams isn't torn down. The PEM is multi-line, so use a heredoc:
   ```
   github_token             = "..."
   cloudflare_api_token     = "..."
   teams_alerts_webhook_url = "https://..."
   github_app_pem           = <<EOT
   -----BEGIN RSA PRIVATE KEY-----
   ...
   -----END RSA PRIVATE KEY-----
   EOT
   ```
5. **`az login`** to the playground subscription, then **run `infra/init/bootstrap.sh`**.
6. **Run init** from `infra/init/`: `terraform init -upgrade -reconfigure -backend-config=../terraform-backend/init.tfbackend`, `terraform plan -out tfplan`, `terraform apply tfplan`.
7. **Push to `infra` main** with a change under `cloud/` or `k8s/terraform/`. CI applies `cloud`, then `k8s/terraform`. The first cloud apply may hit 403 on the blob upload while the role assignment propagates; rerun (0006).
8. **Flux.** The k8s stack installs the AKS Flux extension and Flux configurations for `infra`, `backend`, `frontend` over SSH with deploy keys (0003, `infra/k8s/terraform/flux.tf`). Flux then reconciles `infra/k8s/gitops`: cert-manager issues the wildcard cert via DNS-01 on Cloudflare (0010, 0018), Istio gateway, observability, oauth2-proxy. The Flux UI is at `https://flux.hienvuong.com`, behind oauth2-proxy (Entra login, app registration in `infra/cloud/oauth2-proxy.tf`).
9. **App images.** Push to `backend` and `frontend` main to build images into the new ACR; the workflow opens and merges the deploy PR and Flux rolls it out (0004, 0011).
10. **Check.** Log in to Grafana (role from `Hien Test Group Admin`) and the Flux UI.

## Limits

- The preflight only knows the rights listed in it. A new kind of right that init needs must be added to the script, or the "fail early" guarantee is gone.
- The resource provider list is maintained by hand against the resource types in code.
- Not verified by an actual rebuild from zero; the steps are derived from the code and the docs below.

## References

- https://github.com/hashicorp/terraform-provider-azurerm/blob/main/website/docs/index.html.markdown (`resource_provider_registrations`)
- https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-use-gitops-flux2 (resource providers for Flux)
- https://learn.microsoft.com/en-us/graph/api/user-list-transitivememberof
- https://docs.github.com/en/rest/orgs/members#get-an-organization-membership-for-the-authenticated-user
