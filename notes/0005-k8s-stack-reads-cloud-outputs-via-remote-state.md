# 0005 – k8s Terraform stack reads cloud outputs via remote state

## Context

Infra Terraform is split into two roots: `infra/cloud` (Azure resources, AKS) and `infra/k8s/terraform` (Flux config on the cluster). The split exists only because of the provider chicken-and-egg problem: Kubernetes/Helm providers need the cluster to exist first, so cluster and in-cluster resources are applied separately. The same person (and the same CI identity) owns both.

The k8s stack found cloud resources through `data` sources that rebuilt their names (`id-${project_name}-loki`, `rg-${project_name}`, …). Key Vault and Service Bus names include `random_string.suffix` from the cloud stack, so they were hardcoded (`kv-playground-gw2y5h`). Copied by hand, these can drift.

## Decision

**The cloud stack publishes what k8s needs as outputs, and k8s reads them with `terraform_remote_state`.**

- `infra/cloud/output.tf` exposes names, IDs, client IDs (map `identity_client_ids`) and the Azure Monitor OTLP endpoints. Those endpoints are now composed in `infra/cloud/monitor.tf` from the real DCE/DCR resources.
- `infra/k8s/terraform/other.tf` holds the `terraform_remote_state "cloud"` block plus lookups for things neither stack creates (Entra groups). `local.cloud` in `infra/k8s/terraform/locals.tf` is shorthand for the outputs. `flux.tf` uses `local.cloud.*`.
- Things the cloud stack only looks up (manual RG, DNS zone) are passed through as outputs too, so k8s has a single source.

Options considered (from the HashiCorp remote state doc):

- **Keep `data` lookups by name**: common and fine for convention names, but doesn't solve the random suffix and duplicates the naming convention in both stacks.
- **Publish to a separate store** (Automation variable, blob, ConfigMap…), which HashiCorp recommends: it exists so consumers don't need read access to the whole state. Here both stacks are owned by the same person and identity, so it adds a resource and a moving part for no gain.
- **Remote state only for the suffix**: minimal, but k8s would still rebuild every other name.
- **Drop the random suffix**: renames Key Vault and Service Bus, which means recreation (and Key Vault soft-delete). Not worth it.

## Consequences

- The k8s stack needs read access to `cloud.tfstate`, i.e. to everything in it, secrets included. That is acceptable because the CI identity already has Storage Blob Data Contributor on the state account.
- Outputs are now a contract between the stacks: renaming/removing one breaks k8s at plan time.
- k8s only sees outputs from the last cloud **apply**. Order cloud → k8s is already enforced in `infra/.github/workflows/deploy.yml` (`needs: terraform-cloud`).
- The remote state `config` repeats `terraform-backend/main.tfbackend` (a `.tfbackend` file can't be referenced from HCL).
- Verified: the first CI run after the change showed k8s plan "No changes", so every Flux substitution value stayed identical.

## References

- https://developer.hashicorp.com/terraform/language/state/remote-state-data
- https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs (section on separating cluster creation from Kubernetes resources)
