---
myst:
  html_meta:
    description: "How to migrate to the new Livepatch Server machine charm with Livepatch on-prem."
---

(server-how-to-guides-migrate-to-the-new-machine-charm)=

# Migrate to the new Livepatch Server machine charm

This guide describes how to migrate from the outdated Livepatch Server machine charm to the new and improved livepatch-server machine charm. The destination uses the existing PostgreSQL application and database. There is no need to deploy a replacement database or copy its contents during the migration.

## Identify the source charm

Run `juju status --relations` and identify the source charm:

| Source charm type | Charm name | Channel | Configuration |
| --- | --- | --- | --- |
| Reactive machine charm | `canonical-livepatch-server` | `latest/*` | Underscore-style options, such as `patch_sync_enabled` |
| Original Operator machine charm | `canonical-livepatch-server` | `ops1.x/*` | Dotted options, such as `patch-sync.enabled` |

The previous `emit-updated-config` action has been removed; update existing automation to use `migrate-reactive-config` for reactive sources or `migrate-ops-config` for original Operator sources.

The examples use `livepatch` for the old application, `canonical-livepatch` for the new application, `postgresql` for the existing database application, and `haproxy` for the proxy. Replace names, unit numbers, and placeholders with your deployment's values. Run commands in the intended Juju model. The new charm will be in the `latest` channel, with the `ops` channel being deprecated.

```{warning}
Schedule a maintenance window. The destination upgrades the existing database schema automatically. Disconnect the old application from the proxy and PostgreSQL, and verify that it can no longer write before allowing the upgrade. Do not reconnect it to the upgraded schema. Rollback requires a pre-migration database backup and, where applicable, a matching patch-storage backup.
```

## Prerequisites

This guide assumes:

* A healthy source deployment with administrator access and a dedicated livepatch database provided by a PostgreSQL machine charm.
* Juju with support for the destination charm and Juju secrets, and machines with a supported base and architecture.
* Compatible PostgreSQL client tools, `jq`.
* PostgreSQL administrator access and a tested backup and recovery procedure.
* Access to existing credentials and any required Ubuntu Pro entitlement, certificates, storage, and monitoring integrations.

The database handover below uses the `database` endpoint and `postgresql_client` interface. Check endpoints using `juju status --relations` and `juju show-application`. Identify other database clients and writers.

A reactive source may use a legacy PostgreSQL relation. Configuration conversion does not convert that relation or guarantee schema compatibility. Confirm support for the existing provider and source schema, and review the legacy relation's database/user lifecycle before following the database handover.

The destination requests the database named `livepatch`. If the source uses another name, arrange a supported migration path first. The conversion action does not preserve `psql_dbname` as a destination option.

## Prepare the new application

Deploy the approved charm alongside the source:

```bash
juju deploy canonical-livepatch-server canonical-livepatch --channel latest/stable \
  --base ubuntu@22.04 --constraints arch=amd64
```

Adjust the base for the release. Do not relate the destination to PostgreSQL or HAProxy yet. A blocked status reporting the missing URL is expected. In the destination charm covered here, configuration validation precedes automatic schema upgrades. Any nonempty URL, even a placeholder, releases this gate. Do not run `schema-upgrade` during preparation.

Confirm this ordering for the destination release. This preparation gate is for a newly deployed destination; do not rely on clearing the URL to isolate an application that already has database access.

Save exports in a restricted directory because they can contain credentials:

```bash
umask 077
mkdir livepatch-migration
cd livepatch-migration
juju status --relations
juju config livepatch --format=json > old-config.json
juju config canonical-livepatch --format=json > new-options.json
juju config haproxy --format=json > old-haproxy-config.json
```

Record charm and snap revisions, relations, exposure, unit counts, and any unit-specific settings as well.

## Migrate configuration

Both actions accept the contents of a `juju config` YAML or JSON export in `config-file`, not a filename. Exports may contain options or settings mapping. The actions return a configuration yaml payload without applying it, creating secrets, changing relations, or migrating the database.

The commands below use `jq -Rs .` to encode the export as a string-valued action parameter. No parameter file is needed. This prevents Juju from interpreting the document as a mapping rather than a string. Do not use an unencoded `config-file="$(cat old-config.json)"`.

```{note}
Juju retains action parameters and results in action history. Exports and converted results can contain credentials. Follow your credential-handling policy; where necessary, remove sensitive entries from the conversion input and migrate them separately through secrets. Deleting local files does not erase action history.
```

### Operator source

Run the Operator migration action and write the converted configuration directly:

```bash
juju run canonical-livepatch/leader migrate-ops-config config-file="$(jq -Rs . old-config.json)" --format=json | jq -er '.[] | select(.status == "completed") | .results.result["new-config"]' > new-config.yaml
```

The action extracts dotted options supported by the destination charm's config definitions. It preserves value types, including `false`, zero, empty strings, and exported defaults, while omitting unset/null values. It does not substitute destination defaults or disable synchronization automatically.

Known obsolete keys appear in `removed-keys`; other unsupported keys appear in `unrecognized-keys`. Review them against the source export before discarding any feature:

| Options | Migration requirement |
| --- | --- |
| `auth.sso.*` | SSO logins are deprecated, use http basic auth. |
| `cloud-delay.enabled`, `cloud-delay.default-delay-hours` | Resolve dependencies on cloud-based rollout delays; there are no equivalent charm options. |
| `machine-reports.event-bus.*` | Kafka/event-bus reporting is not migrated. Database-backed reports do not replace downstream consumers. |

Continue with [Extract and review the result](#extract-and-review-the-result).

### Reactive source

For a reactive source, run this command instead:

```bash
juju run canonical-livepatch/leader migrate-reactive-config config-file="$(jq -Rs . old-config.json)" --format=json | jq -er '.[] | select(.status == "completed") | .results.result["new-config"]' > new-config.yaml
```

It translates underscore-style options, renames legacy values where necessary (for example, file storage to filesystem), and derives feature flags from the source settings. Removed and unrecognized keys are reported separately.

### Extract and review the result

The command writes the converted configuration to `new-config.yaml` in one step. Use `juju show-task <TASK_ID>` with the task ID printed by `juju run` to check the completed status and review `result.removed-keys` and `result.unrecognized-keys`; no separate extraction step is needed.

Do not proceed on failure. Malformed exports and recognized options from the wrong source format are rejected rather than silently dropped. Correct the input or select the appropriate action; do not combine source formats.

Both actions return YAML nested under the actual destination application name. No manual top-level rename is needed:

```yaml
canonical-livepatch:
  patch-storage.type: postgres
  patch-sync.enabled: true
  server.burst-limit: 500
```

Review every removed and unrecognized key; Juju may render those lists as strings. Reactive migration removes options such as `psql_dbname` and `psql_roles`. Unrecognized `filestore_path` or `port` may need manual translation to `patch-storage.filesystem-path` or `server.server-address`. Resolve removed SSO, cloud-delay, and event-bus features rather than silently discarding them. Compare the result with `new-options.json` before applying it.

## Move credentials to Juju secrets

Create `basic-users.secret.yaml` containing the existing bcrypt hashes:

```yaml
value: 'admin:<EXISTING_BCRYPT_HASH>'
```

Create and grant the secret:

```bash
chmod 600 basic-users.secret.yaml
juju add-secret livepatch-basic-users --file basic-users.secret.yaml
juju grant-secret livepatch-basic-users canonical-livepatch
```

Set `auth.basic.users-secret` in `new-config.yaml` to the returned URI, remove `auth.basic.users`, and set `auth.basic.enabled: true` if appropriate. Do not hash existing hashes again.

Other supported mappings include:

| Plain options | Secret option | Content keys |
| --- | --- | --- |
| `patch-sync.token` | `patch-sync.token-secret` | `value` |
| `patch-storage.swift-api-key` | `patch-storage.swift-api-key-secret` | `value` |
| `influx.token` | `influx.token-secret` | `value` |
| `contracts.user`, `contracts.password`, `contracts.ca` | `contracts.credentials-secret` | `user`, `password`, `ca` |
| `patch-storage.s3-access-key`, `patch-storage.s3-secret-key` | `patch-storage.s3-credentials-secret` | `access-key`, `secret-key` |

Grant each secret to the destination. Keep contracts CA content base64-encoded. Omitted grouped-secret keys retain their plain configuration values, so review remaining fallbacks. Secret URIs do not transfer between models.

The reviewed file should have this structure:

```yaml
canonical-livepatch:
  server.url-template: "https://livepatch.example.com/v1/patches/{filename}"
  server.server-address: "0.0.0.0:8080"
  server.is-hosted: false
  patch-storage.type: postgres
  patch-sync.enabled: false
  auth.basic.enabled: true
  auth.basic.users-secret: "secret:<SECRET_ID>"
```

This example is only a subset. Retain other applicable settings. Do not apply the file until database and storage checks are complete.

## Migrate database access

The database stays on the existing PostgreSQL application. The client relation, relation user, application-object ownership, and schema change.

### Identify the old database relation username

Do this before removing the old PostgreSQL relation. The username belongs to the database credentials issued for the old Livepatch application; it is not the application name or the PostgreSQL administrator username.

Display the old Livepatch unit's database credential references without printing passwords:

```bash
juju show-unit livepatch/0 --format=json \
  | jq '.[]."relation-info"[] |
      select(.endpoint == "database") | ."application-data" |
      {username, "secret-user": ."secret-user"}'
```

Replace `livepatch/0` with an actual old application unit. If `username` is populated, use that value. Otherwise, copy the displayed `secret-user` URI into this command:

```bash
juju show-secret <OLD_SECRET_USER_URI> --reveal --format=json \
  | jq -er '.[].content.Data.username'
```

Note the printed username before removing the relation. For example, if it prints `relation-28`, use `\set old_livepatch_user 'relation-28'` in the later SQL. Replace `<OLD_RELATION_USER>` with that actual value. Do not derive the username from a Juju relation ID: provider naming schemes vary.

### If the relation has already been removed

Use the username you noted before removal. If it is unavailable, connect to the database as its administrator and identify surviving connections from the old unit addresses:

```psql
SELECT DISTINCT usename, client_addr
FROM pg_stat_activity
WHERE datname = current_database()
  AND client_addr = '<OLD_LIVEPATCH_UNIT_IP>'::inet;
```

Repeat for each old unit, using its address from `juju status`. Verify that the returned role belongs to the old Livepatch application. If the query returns no sessions, it does not identify the old role: use saved credential records or investigate with the database administrator rather than guessing from role names or table ownership.

Once verified, use that username in the `\set old_livepatch_user` command below. Do not recreate a relation just to discover its old username; a new relation can issue a different one.

### Disconnect the old application

Begin the maintenance window. Drain client and administrative traffic, stop scheduled administrative jobs, and freeze other storage writers. Remove both the old application's proxy relation and its PostgreSQL relation:

```bash
juju remove-relation livepatch:website haproxy:reverseproxy
juju remove-relation livepatch:database postgresql:database
juju status --relations
```

Wait until both relations have disappeared and their removal hooks have completed. Keep PostgreSQL, its units, and its storage. Removing the `livepatch:database` to `postgresql:database` relation disconnects both ends; there is no second reverse-direction relation to remove.

Keep ingress drained until verification passes. Relation changes can reconfigure or restart the proxy. Removing a proxy relation does not block direct access to the old units, so preserve the maintenance access restrictions.

### Verify that old writers are disconnected and back up the database

Identify the PostgreSQL primary and address using `juju status`. Check the provider's credential interface:

```bash
juju actions postgresql
```

Where supported, obtain administrator credentials with:

```bash
juju run <POSTGRESQL_PRIMARY_UNIT> get-password
```

For releases exposing credentials through secrets instead, including PostgreSQL 16 charm releases without `get-password`, discover provider secrets:

```bash
juju secrets --owner application-postgresql
```

Use that release's documented administrator secret and credential keys. Do not confuse it with a restricted client relation account. Protect revealed credentials; retrieval depends on the provider release, not just the database major version.

Before taking the backup, connect as the PostgreSQL administrator:

```bash
psql -h <POSTGRESQL_IP> -U operator -d livepatch -W
```

Set `old_livepatch_user` to the username obtained in [Identify the old database relation username](#identify-the-old-database-relation-username), then check the old role and sessions:

```psql
\set ON_ERROR_STOP on
\set old_livepatch_user '<OLD_RELATION_USER>'
\echo :old_livepatch_user

SELECT rolname, rolcanlogin
FROM pg_roles WHERE rolname = :'old_livepatch_user';

SELECT usename, client_addr, state
FROM pg_stat_activity
WHERE datname = current_database() AND usename = :'old_livepatch_user';
```

Use the discovered relation username, not a hard-coded example. An idle session remains connected and can issue more queries.

Some PostgreSQL provider versions retain the old role and existing sessions after relation removal. If the old dedicated relation role still exists, disable new logins and terminate its remaining sessions in this database:

```psql
ALTER ROLE :"old_livepatch_user" NOLOGIN;

SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = current_database()
  AND usename = :'old_livepatch_user'
  AND pid <> pg_backend_pid();
```

Skip these statements if the provider has already removed the role. Repeat the role and session queries: the role must be absent or unable to log in, and no old-role sessions should remain. Confirm this is the old application's dedicated role before changing it; do not disable a shared or administrator account.

Also prevent the old application and scheduled jobs from writing to separate patch storage. Database disconnection alone does not revoke independently configured Swift, S3, or filesystem access. Keep those stores isolated from old writers throughout the migration.

Exit with `\q` once these checks pass. Avoid configuration changes or actions that could reconnect or reactivate old writers during the migration.

Use backup and restore tools from the same PostgreSQL major version as the database. Check `pg_dump --version` and `pg_restore --version` before continuing; the client installed by the operating system may be newer than the server. A newer client can create an archive that introduces restore statements the older server cannot execute.

Back up the database:

```bash
pg_dump -h <POSTGRESQL_IP> -U operator -d livepatch \
  --format=custom --file=livepatch-before-migration.dump
pg_restore --list livepatch-before-migration.dump
sha256sum livepatch-before-migration.dump
```

### Relate the new application to the existing database

The old PostgreSQL relation was removed before the backup; do not remove it again. After verifying the backup and that old writers remain disconnected, relate the destination to the same PostgreSQL application:

```bash
juju relate canonical-livepatch:database postgresql:database
```

Confirm the relation requests and supply the existing livepatch database.

### Identify the new PostgreSQL relation user

The Juju application name is not the database username:

```bash
juju show-unit canonical-livepatch/0
```

Find `secret-user` in the database relation's application data and read its username:

```bash
juju show-secret <SECRET_USER_URI> --reveal --format=json \
  | jq -er '.[].content.Data.username'
```

Some provider versions expose `username` directly in relation data. Inspect the provider unit's relation data if necessary.

The role might be `relation-28` or `relation_id_22`. Names depend on the provider and relation. Recreating a relation can issue a new username and password even when the application name is unchanged. Discover the role from the new relation; quote hyphenated PostgreSQL identifiers.

### Check application-object ownership

Connect as the administrator:

```bash
psql -h <POSTGRESQL_IP> -U operator -d livepatch -W
```

Run these read-only ownership queries and compare the owners with the exact username discovered from the new relation.

```psql
SELECT datname, pg_get_userbyid(datdba) AS owner
FROM pg_database WHERE datname = current_database();

SELECT tablename, tableowner
FROM pg_tables WHERE schemaname = 'public';

SELECT sequencename, sequenceowner
FROM pg_sequences WHERE schemaname = 'public';

SELECT nspname, pg_get_userbyid(nspowner) AS owner
FROM pg_namespace WHERE nspname = 'public';

SELECT typname, pg_get_userbyid(typowner) AS owner
FROM pg_type
WHERE typnamespace = 'public'::regnamespace AND typtype = 'e';

SELECT p.oid::regprocedure AS function, pg_get_userbyid(p.proowner) AS owner
FROM pg_proc p
WHERE p.pronamespace = 'public'::regnamespace
  AND NOT EXISTS (
    SELECT 1 FROM pg_depend d
    WHERE d.classid = 'pg_proc'::regclass
      AND d.objid = p.oid AND d.deptype = 'e'
  );
```

The destination user must own the application objects it needs to alter and have the necessary schema privileges. Providers may transfer tables and sequences but leave enums under `operator` or the old relation user. If the schema owner is `pg_database_owner`, check the database owner and effective privileges rather than treating that owner as an error.

If these application enums exist and are still owned by the old user or `operator`, transfer them before allowing the schema upgrade. Replace `<NEW_RELATION_USER>` in each statement with the exact username discovered above, retaining the double quotes. For example, a role named `relation-18` must be written as `"relation-18"`, not `"relation_18"`.

```psql
BEGIN;
ALTER TYPE public.job_status OWNER TO "<NEW_RELATION_USER>";
ALTER TYPE public.job_type OWNER TO "<NEW_RELATION_USER>";
ALTER TYPE public.patch_state OWNER TO "<NEW_RELATION_USER>";
COMMIT;
```

Application functions also matter, particularly after restoring a backup without ownership. If the function query lists the following Livepatch functions under another owner, transfer them in the same administrator session:

```psql
BEGIN;
ALTER FUNCTION public.bakery_rootkeys_expire_func() OWNER TO "<NEW_RELATION_USER>";
ALTER FUNCTION public.comparable_patch_version(text) OWNER TO "<NEW_RELATION_USER>";
COMMIT;
```

Only run these statements for functions confirmed to exist. The query excludes extension-owned functions, which must not be reassigned as application objects. Complete ownership changes before starting the workload; an authentication initialization failure can persist until the service is restarted. Repeat the ownership queries, then exit with `\q`.

## Migrate patches

### PostgreSQL

For patches in the main livepatch database, no copy is needed. Keep `patch-storage.type: postgres` and leave `patch-storage.postgres-connection-string` unset; the relation supplies it.

For a separate PostgreSQL patch database, back it up and verify destination access and schema requirements. Retain its connection string only if the credentials remain valid after removing old relations.

### Swift and S3

Retain the container or bucket, endpoint, region, and credentials. Verify connectivity from new units. No file copy is needed for the same store, but keep a snapshot or versioned backup and prevent concurrent old/new writers.

### Filesystem

Copy the patch directory to every new unit serving downloads. Preserve permissions and compare counts and checksums. The snap default is:

```text
/var/snap/canonical-livepatch-server/common/patches
```

Check for a custom source path and set `patch-storage.filesystem-path` accordingly. The snap must be able to access it. The action does not copy files, and the charm does not replicate filesystem storage between units.

## Start and verify the new server

After backup, ownership, and storage checks, apply the reviewed file:

```bash
juju config canonical-livepatch --file new-config.yaml
juju status --relations
```

The charm upgrades the existing schema and starts the workload. A manual `schema-upgrade` action is not normally needed. If migration fails, keep the old server disconnected: earlier upgrade steps may already have committed.

Adjust the port for `server.server-address`. Inspect schema version, database and patch-store health, and enabled functions, not just HTTP 200 or Juju status. Verify required features, including patch blocklisting, against source behavior.

Restore entitlement, contracts, monitoring, and subordinate integrations. Where online activation uses `enable`, run it on each new unit using the approved token-handling procedure. Action parameters remain in Juju history.

## Switch traffic

After direct checks pass, connect the new application to the existing proxy:

```bash
juju relate canonical-livepatch:website haproxy:reverseproxy
```

The old proxy relation was removed during maintenance; do not remove it twice. HAProxy consumes backend data through `reverseproxy`, not `website`. The database relation is already established and need not be added again.

Keep TLS certificates, hostname, routing restrictions, and firewall rules. Confirm generated backends contain only new units and validate the proxy configuration before reopening traffic.

Only after validation, remove the old application:

```bash
juju remove-application livepatch
```

Keep PostgreSQL and its storage: the new application now uses them. Retain pre-migration backups under the backup policy.

## Roll back

Reconnecting the old server to the upgraded schema is not a rollback. Drain traffic and remove the new application's proxy and database relations:

```bash
juju remove-relation canonical-livepatch:website haproxy:reverseproxy
juju remove-relation canonical-livepatch:database postgresql:database
```

Remove only relations that still exist. Record the new application's relation username before removal. Use the same role and session checks described above to ensure it cannot reconnect or rerun schema migrations during recovery. Keep external patch storage isolated from new writers as well.

Restore the pre-migration database and any matching storage snapshot using the provider's recovery procedure. Preserve the failed migration state for investigation. Writes after the original backup will not be in that backup.

Do not ignore restore errors. For example, an archive of PostgreSQL 14 made by a PostgreSQL 18 client may emit `SET transaction_timeout = 0`, which PostgreSQL 14 does not support. Prefer matching-major tools when taking the backup. If such an archive already exists, inspect its generated SQL and test recovery in an isolated database first. If that session setting is the only incompatibility, this narrowly scoped restore pipeline omits it:

```bash
set -o pipefail
pg_restore --no-owner --no-privileges --file=- livepatch-before-migration.dump \
  | sed '/^SET transaction_timeout = 0;$/d' \
  | psql -X --single-transaction -v ON_ERROR_STOP=1 \
      -h <RESTORE_POSTGRESQL_IP> -U operator -d <EMPTY_RESTORE_DATABASE>
```

This example requires a prepared empty database, a compatible `psql` client, and a restricted password file for noninteractive authentication. It restores objects under the administrator, so ownership must be assigned to the actual relation user before starting Livepatch. It is a recovery example, not a step to copy the database during the same-instance migration.

Restore the old application's database relation while keeping client and administrative traffic isolated. Discover the newly issued user and verify ownership of restored tables, sequences, schema, enums, and application functions. Do not restore old connection strings blindly; relation credentials may have changed.

Restore old configuration, effective authentication, and required integrations. Verify the old workload against the restored schema before reconnecting the proxy and returning traffic.
