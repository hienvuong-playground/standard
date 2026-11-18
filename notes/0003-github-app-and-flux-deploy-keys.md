# 0003 – GitHub App for Terraform, deploy keys for Flux

## Context

Two different things need to talk to GitHub:

- Terraform (stack `infra/k8s/terraform`) must manage GitHub resources, currently the repo deploy keys.
- Flux in AKS must pull the `backend` and `infra` repos over SSH (GitOps).

Wanted: no personal PAT, and as little manual secret handling as possible.

## Decision

**GitHub App for Terraform.** The `github` provider authenticates with `app_auth` (App ID, installation ID, PEM) instead of a PAT, so it isn't tied to a person. See `infra/k8s/terraform/providers.tf`. The PEM comes from `var.github_app_pem`: local file `secrets.auto.tfvars` (gitignored), or `TF_VAR_github_app_pem` from the `GH_APP_PEM` Actions secret in CI (`infra/.github/workflows/deploy.yml`).

**Deploy keys for Flux.** One read-only ED25519 key per repo (`backend`, `infra`). Terraform generates the pair (`tls_private_key`), registers the public key (`github_repository_deploy_key`) and hands the private key to the Flux configuration. See `infra/k8s/terraform/flux.tf`. Deploy keys are scoped to a single repo and read-only, so a leaked key has a small blast radius.

**Manual vs code:**

| Item | How |
|---|---|
| GitHub App + its PEM | Manual: org settings → Developer settings → GitHub Apps. No webhook. Generate a private key |
| `GH_APP_PEM` Actions secret | Code: org secret visible to `infra` only, written by `infra/init` from `init/secrets.auto.tfvars` (see 0015) |
| Deploy key pairs, public key on GitHub, private key into Flux | Code (Terraform) |

**App permissions.** Only repository permission `Administration: Read and write`, which is what managing deploy keys needs. Installed on `backend`, `frontend` and `infra`. App ID and installation ID are hardcoded in `infra/k8s/terraform/providers.tf`, so a new App means editing them.

Rejected: automating the App itself. Terraform authenticates *through* the App, so it can't create it (chicken-and-egg). Manual is fine for a one-time setup.

## Consequences

- Rotating the App key: update `init/secrets.auto.tfvars` and rerun init (plus `k8s/terraform/secrets.auto.tfvars` if planning locally).
- Deploy private keys live in Terraform state, so the state must stay protected.
- Only the `k8s` stack uses the App; the `github` provider block in `infra/cloud/providers.tf` is commented out.
- Verified by reading the Terraform config; no apply was done for this note.

## References

- https://registry.terraform.io/providers/integrations/github/latest/docs#github-app-installation
- https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys
