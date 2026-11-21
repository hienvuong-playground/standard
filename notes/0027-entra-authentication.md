# 0027 – How Entra authentication works here

## Context

CI signs in with `azure/login`, and the CI identity has `Grafana Admin` on AMG. Still, Terraform needed an explicit `az account get-access-token` step before the Grafana provider could call AMG (0026). This note explains why, and the general mechanism behind every Entra login in this setup: CI, pods, local `az`.

## The pieces

**Identity (principal).** Who is acting. Here:

- Users, e.g. you after `az login`.
- Groups (`Hien Test Group Admin` / `Dev`). They don't sign in; they only collect members for role assignments.
- Managed identities:
  - CI: user-assigned `id-playground` (`infra/init/main.tf`).
  - Backend pod: `id-playground-backend` (`infra/cloud/id.tf`).
  - AMG: system-assigned (`infra/cloud/managed-grafana.tf`).

**Sign-in.** Proving to Entra that you are that identity. Afterwards the tool holds a session, not yet a token for any service.

- User: browser login via `az login`.
- CI, federated credential:
  - GitHub issues an OIDC token for the job: issuer `token.actions.githubusercontent.com`, subject `repo:…/infra@…:ref:refs/heads/main`.
  - Entra trusts that issuer+subject for `id-playground` and exchanges the token. The audience of the exchange is `api://AzureADTokenExchange`.
- Pod, workload identity: the same exchange. The AKS OIDC issuer signs the pod's service account token, with subject `system:serviceaccount:backend:backend-workload`.

No secret is stored for either. Trust comes from the issuer + subject pair.

**Access token.** A short-lived JWT that Entra issues for one service. Claims that matter:

- `aud`: which service it is for.
- `oid`: the caller's object id.
- `tid`: the tenant.
- `exp`: expiry, about 60–90 minutes.

It is requested per service, as `--resource <id>` or scope `<id>/.default`.

## One token per service (audience)

A service accepts only tokens whose `aud` is itself. A token for ARM is useless at AMG, and AMG returns 401.

| Service | Audience | Used by |
|---|---|---|
| ARM (control plane) | `https://management.azure.com` | `azurerm`, `az` |
| Microsoft Graph | `https://graph.microsoft.com` | `azuread` |
| Storage | `https://storage.azure.com` | Terraform state backend |
| AMG Grafana API | `ce34e7e5-485f-4d76-964f-b3d2b16d1e4f` | Grafana provider (0026) |
| AKS API server | `6dae42f8-4368-4678-94ff-3960e28e3630` | `kubectl` via `kubelogin` |
| Postgres | `https://ossrdbms-aad.database.windows.net` | backend (`backend/src/db/pool.js`) |

## Who fetches the token

Something always has to request a token for the right audience. Tools differ only in whether they do it for you:

- **Azure-aware tools fetch it themselves.**
  - `azurerm`, `azuread` and the `azurerm` backend use Azure CLI auth when no `ARM_*` variables are set, and call `az` per audience.
  - The backend uses `DefaultAzureCredential` and calls `getToken(<postgres scope>)` for each new connection. In the pod, it finds the workload identity environment variables.
- **Tools that know nothing about Azure must be handed a token.**
  - Grafana provider: the workflow puts the token in `GRAFANA_AUTH`.
  - `kubectl`: the `kubelogin` exec plugin fetches it.
  - Postgres: the token is sent as the password.

## Token says who, role says what

These are two separate checks, made by the target service on every request:

1. **Authentication.** It verifies the token's signature and `aud`, and reads `oid`.
2. **Authorization.** It looks up what that `oid` may do. For most Azure services this is an Azure RBAC role assignment on the resource. Postgres instead maps the Entra principal to a database role with the same name (0017).

Without a token, the service doesn't know who is calling. With a token but no role, it knows the caller and refuses.

**Control plane and data plane are separate permissions.**

- ARM operations (create AKS, assign roles) are `Actions`. `Owner` and `Contributor` grant those.
- Reading data inside a service is `DataActions`, which those roles don't include. Examples:
  - `Grafana Admin` / `Viewer`.
  - `Monitoring Data Reader` for PromQL on AMW.
  - `Azure Service Bus Data Owner`.

That is why `Owner` on the subscription doesn't get you into AMG (0021).

## Example: Terraform calling AMG in CI

```
azure/login            GitHub OIDC token ──exchange──▶ Entra: session as id-playground
az get-access-token    session ──▶ Entra: token, aud = ce34e7e5…, oid = id-playground
Grafana provider       Authorization: Bearer <token> ──▶ AMG
AMG                    checks signature + aud → reads oid → Azure RBAC: Grafana Admin → allowed
```

The `cloud` stack goes through the same steps for ARM, Graph and Storage, but the providers run them internally.

## References

- https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens
- https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation
- https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview
- https://learn.microsoft.com/en-us/azure/role-based-access-control/role-definitions#control-and-data-actions
- https://learn.microsoft.com/en-us/azure/developer/javascript/sdk/authentication/credential-chains
- https://learn.microsoft.com/en-us/azure/managed-grafana/how-to-api-calls
- https://azure.github.io/kubelogin/concepts/aks.html
