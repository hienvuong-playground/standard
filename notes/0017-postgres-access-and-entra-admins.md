# 0017 – Postgres: open firewall, and why removing an Entra admin can hang

## Context

Two things came up while moving the Postgres Entra admin from the `DevOps Engineers` group to `Hien Test Group Admin` (0015):

- The server only allowed Azure services (`allow-azure-services`, `0.0.0.0`–`0.0.0.0`, which in Azure means "Azure services", not "everyone"), so it couldn't be reached from the laptop to inspect or fix anything.
- CI hung for 30 minutes on deleting the old admin, then failed with `polling after AdministratorsMicrosoftEntraDelete: context deadline exceeded`.

## Decision

**Firewall open to all IPs.** `allow-all` (`0.0.0.0`–`255.255.255.255`) in `infra/cloud/postgres.tf`. Password auth is off (`password_auth_enabled = false`), so a connection still needs an Entra token for a principal mapped to a Postgres role. The home IP is dynamic, so a per-IP rule would keep breaking. Rejected: a temporary per-IP rule each time (manual, easy to forget to remove), and a `psql` pod in AKS (more steps for every check).

**Removing an Entra admin: clear what its role owns first.** Deleting an admin makes Azure `DROP ROLE` the matching Postgres role. If that role owns anything, Postgres refuses (`2BP01 ... cannot be dropped because some objects depend on it`) and Azure fails the operation after ~30 s. The azurerm provider doesn't notice and keeps polling until its 30-minute timeout. The real error is only in the Azure activity log (`AadAuthPrincipalDropFailed`).

Here the only dependency was ownership of the `storefront` database. Fixed by hand, connected as a member of the old group:

```sql
ALTER DATABASE storefront OWNER TO "id-playground-backend";
```

To check before removing an admin, in each database (`postgres`, `storefront`):

```sql
SELECT classid::regclass, pg_describe_object(classid, objid, objsubid), deptype
FROM pg_shdepend WHERE refobjid = (SELECT oid FROM pg_roles WHERE rolname = '<role>');
```

If it owns more than a database: `REASSIGN OWNED BY "<role>" TO "<other role>"; DROP OWNED BY "<role>";`. The target role must already exist, and the new admin's role is only created after the old one is dropped (Terraform replaces destroy-first), so reassign to a role that stays (e.g. `id-playground-backend`).

## Consequences

- The Postgres endpoint is reachable from the internet; protection is Entra auth only.
- `storefront` is now owned by the backend identity. Not in Terraform (the database resource has no owner argument).
- A hung "deleting Administrator" in CI = check the activity log first, not the timeout.
- A cancelled CI apply can leave the state blob leased; `terraform force-unlock <id>` (lock ID is in the next run's error) once no run is active.
- Verified: after the `ALTER`, `pg_shdepend` showed no dependencies left for `DevOps Engineers`.

## References

- https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-manage-azure-ad-users
- https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-firewall-rules
- https://www.postgresql.org/docs/current/role-removal.html
