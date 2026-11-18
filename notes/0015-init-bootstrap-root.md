# 0015 – The `init` root: bootstrap what would otherwise be manual

## Context

Some things have to exist before CI can provision anything: a place for Terraform state, an identity CI can log in as, and the permissions that identity needs. CI can't create these for itself. The goal is to rebuild the playground from scratch with as few manual steps as possible.

Some permissions can never be put in code, because something has to grant them first. Examples are Owner on the Azure subscription, an Entra admin role, and admin on the GitHub org. These are the true starting point, and a human holds them.

## Decision

A separate Terraform root, `infra/init/`, is run by hand from a laptop by someone who already holds those manual permissions. It turns them into everything CI needs, so the manual part shrinks to "have the top-level rights, run init once". The ordered rebuild steps, the manual pieces and `bootstrap.sh` are in 0016.

Rights the runner must already hold (`bootstrap.sh` checks all of them first):
- Azure: `Owner` on the subscription, to create the CI identity's role assignments.
- Entra: `Global Administrator`, to assign the `Application Administrator` / `Directory Readers` directory roles and create groups.
- GitHub: admin of the org, with a classic PAT scoped `admin:org` (org variables, secrets, Actions settings) and `repo` (repo-level Actions settings).
- Cloudflare: an account-owned API token `infra-ci`, made by hand in the dashboard. Its permissions and why it can't come from code are in 0018.

What it creates (`infra/init/main.tf`):
- The resource group, storage account and `tfstate` container that every other stack uses as its backend (`infra/terraform-backend/main.tfbackend`).
- One user-assigned identity for CI. It has a federated credential for each repo in `github_repo_ids` (`infra/init/locals.tf`, main branch only), so there are no stored secrets.
- Roles for that identity: `Owner` on the subscription and `Storage Blob Data Contributor` on the state account. On Entra: `Application Administrator` and `Directory Readers`, so the stacks can manage app registrations and look up groups.
- GitHub org variables `AZURE_CLIENT_ID` / `AZURE_TENANT_ID` / `AZURE_SUBSCRIPTION_ID`, visible to all repos, so every workflow can log in.
- GitHub org secret `GH_APP_PEM`, visible to `infra` only (see 0003).
- GitHub org secret `CLOUDFLARE_API_TOKEN` (the hand-made `infra-ci` token), visible to `infra` only (see 0018).
- GitHub Actions workflow permissions, org-level and on `backend`/`frontend`: default `GITHUB_TOKEN` read (every workflow declares its own `permissions:`), and "Allow GitHub Actions to create and approve pull requests" on, which the deploy-PR flow needs (0004). Both levels are managed because it's unclear whether repos inherit the org value.
- Entra security groups `Hien Test Group Admin` (the runner is owner and member) and `Hien Test Group Dev`. Downstream stacks look them up by name: Postgres AAD admin (`infra/cloud/postgres.tf`) and Grafana roles (`infra/k8s/terraform/other.tf`). CI's Entra roles can't create groups, so init does it.

How it runs:
- **State in its own storage account**, created by `bootstrap.sh`, not by Terraform. Init can't keep its state in the account it creates for the other stacks (chicken-and-egg), so one small account sits outside Terraform. Rejected: having the script create the main state account too and init only reading it, which would take that account (used by every stack, and the one more likely to change) out of Terraform: no plan, no drift detection. The init state account is the only unavoidable exception and is not expected to change.
- **Secrets in `init/secrets.auto.tfvars`**, which is gitignored: the PAT, the GitHub App PEM and the Cloudflare token.
- It holds the highest privilege of anything in the setup. Downstream stacks only get what init grants them.

### Deliberate playground shortcuts (not how it should be done for real)

- One CI identity shared by all three repos, with **subscription Owner**. A frontend workflow could do anything in the subscription. Done properly, each repo would get its own identity with least privilege.
- Init secrets (`secrets.auto.tfvars`) live only on one laptop. Lose it and you create a new PAT and App key; nothing is unrecoverable.
- A classic PAT instead of a fine-grained token or a GitHub App.
- The Cloudflare CI token can create tokens, so it effectively has full Cloudflare account access (0018).
- Federated credentials are pinned to `refs/heads/main`. There are no PR or environment subjects, so plan-on-PR wouldn't be able to log in.

These trade a lot of safety for fast setup. That's acceptable only because this is a sandbox.

## Consequences

- Adding a repo that deploys to Azure takes a new entry in `github_repo_ids` (`infra/init/locals.tf`) and a manual init run.
- Init is not in CI. Changes to it are applied by hand, and it's easy to forget that it has drifted.
- Init state is versioned in blob storage; a bad write can be rolled back to an earlier blob version. Deleting `rg-playground-init-state` loses it: the resources still exist, so import them. Nothing downstream stores init state.
- Changing the init state account (e.g. its settings) means editing `bootstrap.sh` and rerunning it; there is no plan.
- Other stacks never reference init outputs. They only see the identity through GitHub variables, and the backend through `main.tfbackend`.

## References

- https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust-user-assigned-managed-identity
- https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-azure
- https://registry.terraform.io/providers/integrations/github/latest/docs/resources/actions_organization_variable
- https://registry.terraform.io/providers/integrations/github/latest/docs/resources/actions_organization_workflow_permissions
- https://registry.terraform.io/providers/integrations/github/latest/docs/resources/workflow_repository_permissions
