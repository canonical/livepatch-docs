---
myst:
  html_meta:
    description: "How to migrate from reactive charm to K8s operator charm with Livepatch on-prem."
---


(server-how-to-guides-migrating-from-the-livepatch-machine-charm-to-the-k8s-charm)=

# Migrate from the Livepatch machine charm to the K8s charm

The Juju framework offered a way to write what were called [reactive charms](https://documentation.ubuntu.com/juju/3.6/reference/charm/#reactive-charm) using the [Reactive](https://charmsreactive.readthedocs.io/en/latest/) framework. Reactive charms have been deprecated and the Livepatch Server reactive charm is no longer actively maintained.

The recommended way of deploying the Livepatch Server is to use the [Kubernetes charm](https://charmhub.io/canonical-livepatch-server-k8s) or the [Livepatch Server snap package](https://snapcraft.io/canonical-livepatch-server). This document describes how to migrate a Livepatch Server instance deployed with a reactive charm to an instance deployed with the Livepatch Server K8s operator.

To determine what charm is deployed in the environment, connect to the Juju controller via SSH, and run `juju status`. Confirm the charm name is "canonical-livepatch-server" and the channel the Livepatch Server was installed from. The output will look like this
![|602x33](/_static/images/wKsSL6h98yOCxXan9YjZ81Dqg.png)

View the table below from the reactive charm to snap migration guide to understand the charm type deployed and the status for that type.

| Charm type | Charm name | Channel | Status/Recommendation |
| --- | --- | --- | --- |
| Machine charm - Reactive | canonical-livepatch-server | latest/\* | Reactive charm (deprecated) |
| Machine charm - Operator | canonical-livepatch-server | ops1.x/\* | Operator charm (deprecated) |
| Kubernetes charm - Operator | canonical-livepatch-server-k8s | latest/\* | Operator charm (recommended for new deployments on Juju) |

## Prerequisites

This guide assumes the following:

- A K8s Juju model.
- Access to the machine charm model.
- The charmed Livepatch K8s operator deployed in a Juju Model.
- A charmed K8s PostgreSQL deployment.
- The IP of the leader PostgreSQL unit for the old Livepatch deployment.
  - To acquire the IP, run `juju status` and look for the public IP under the **Public address** column.
- The `jq` and `yq` utility.
- The `canonical-livepatch-server-admin` snap.

To ensure a clean database migration, do not relate the K8s Livepatch operator to the PostgreSQL charm.

## Migrate configuration

The Livepatch Server operator charms use a simplified and better organized configuration schema. The configuration schema change means the reactive charm configuration is incompatible with the operator charms and needs to be converted.

The K8s operator charms provide an action to convert from the legacy config format to the new format. First, acquire the reactive charm configuration with:

```bash
juju config canonical-livepatch-server > old-config.yaml
```

Next, switch to the Livepatch Server K8s deployment and generate the migrated configuration:

```bash
juju run canonical-livepatch-server-k8s/0 emit-updated-config config-file="$(jq -Rs . old-config.yaml)"
```

Where `old-config.yaml` is the reactive charm configuration.

The output for this action will look like the following:

```yaml
result:
  new-config: |-
    canonical-livepatch-server-k8s:
      auth.sso.url: login.ubuntu.com
      database.connection-lifetime-max: 30m
      database.connection-pool-max: 15
      machine-reports.database.cleanup-interval: 6h
      machine-reports.database.cleanup-row-limit: 1000
      machine-reports.database.retention-days: 90
      patch-cache.cache-size: 128
      patch-cache.cache-ttl: 1h
      patch-cache.enabled: false
      patch-storage.s3-secure: true
      patch-storage.type: postgres
      patch-sync.enabled: true
      patch-sync.flavors: generic,lowlatency,aws
      patch-sync.interval: 1h
      patch-sync.send-machine-reports: false
      patch-sync.upstream-url: https://livepatch.canonical.com
      server.burst-limit: 500
      server.concurrency-limit: 50
      server.log-level: warn
      server.url-template: <your url template>
  removed-keys: '[''psql_dbname'', ''psql_roles'']'
  unrecognized-keys: '[''filestore_path'', ''nagios_context'', ''nagios_servicegroups'',
    ''port'']'
```

The K8s charm compatible configuration is in the `new-config` field, with deprecated and unknown configuration keys listed in `removed-keys` and `unrecognized-keys` respectively.

The whole output for the action can also be taken and the new configuration portion written into a new YAML file with:

```bash
juju run canonical-livepatch-server-k8s/0 emit-updated-config config-file="$(jq -Rs . old-config.yaml)" | yq ".result"."new-config" > config.yaml
```

With the above example, the resulting configuration file will contain:

```yaml
canonical-livepatch-server-k8s:
  auth.sso.url: login.ubuntu.com
  database.connection-lifetime-max: 30m
  database.connection-pool-max: 15
  machine-reports.database.cleanup-interval: 6h
  machine-reports.database.cleanup-row-limit: 1000
  machine-reports.database.retention-days: 90
  patch-cache.cache-size: 128
  patch-cache.cache-ttl: 1h
  patch-cache.enabled: false
  patch-storage.s3-secure: true
  patch-storage.type: postgres
  patch-sync.enabled: true
  patch-sync.flavors: "generic,lowlatency,aws"
  patch-sync.interval: 1h
  patch-sync.send-machine-reports: false
  patch-sync.upstream-url: "https://livepatch.canonical.com"
  server.burst-limit: 500
  server.concurrency-limit: 50
  server.log-level: warn
  server.url-template: <your url template>
```

Now configure the Livepatch K8s operator with the converted reactive charm configuration values:

```bash
juju config canonical-livepatch-server-k8s --file new-config.yaml
```

This only updates configuration values set in the file; any configuration not specified remains at its default.

## Migrate database

> NOTE: This guide assumes the PostgreSQL database was deployed using the machine charm to interact with the Livepatch Server charm. For the Livepatch K8s charm, deploy a K8s PostgreSQL charm in a Juju model (the same model as the Livepatch K8s charm or a different model). In this section, the PostgreSQL 14 charm is deployed on the same model as the Livepatch K8s charm.

### Dump the machine charm database

To migrate the PostgreSQL database from the machine charm environment to the K8s environment, first install the necessary tools:

```bash
sudo apt install postgresql-client postgresql-client-common
```

Next, run the `get-password` action on the PostgreSQL charm:

```bash
juju run postgresql/leader get-password
```

Finally, dump the database using the IP address from earlier:

```bash
pg_dump -Fc livepatch -h <postgres-ip> -U operator > dump-file
```

A data dump of the machine charm PostgreSQL instance is now available. The next steps are to restore the contents on the K8s PostgreSQL charm.

### Restore into the K8s PostgreSQL instance

Before attempting the restoration to the K8s database, ensure the deployment has been scaled down to one unit:

```bash
juju scale-application postgresql-k8s 1
```

> Once the restoration is done, the deployment can be scaled back to however many units are required.

Once the deployment is scaled down, copy the machine charm PostgreSQL dump file into the PostgreSQL container in the leader unit:

```bash
juju scp --container postgresql ./dump-file postgresql-k8s/leader:/tmp/dump-file
```

Next, SSH into the leader unit:

```bash
juju ssh --container postgresql postgresql-k8s/leader bash
```

Finally, restore the database from the dump file:

```bash
pg_restore -h localhost -U operator -d postgres --no-owner --clean --if-exists /tmp/dump-file
```

Once the database has been restored, relate the database to the Canonical Livepatch Server K8s operator and apply the database migrations:

```bash
juju integrate canonical-livepatch-server-k8s:database postgresql-k8s:database
juju run canonical-livepatch-server-k8s/leader schema-upgrade
```

After the restore, the roles need to be updated as the names differ from charm deployments. Specifically, the role that has access to the Livepatch database needs to be given access to each table, since the restore sets all ownership to the `operator` role.

SSH back into the leader unit and log in to `psql`:

```bash
psql -h 127.0.0.1 -U operator -d livepatch-server -W
```

List all users for the `livepatch-server` database:

```text
livepatch-server=# \du
                                                   List of roles
   Role name    |                         Attributes                         |              Member of
----------------+------------------------------------------------------------+--------------------------------------
 admin          | Cannot login                                               | {pg_read_all_data,pg_write_all_data}
 backup         | Superuser                                                  | {}
 monitoring     |                                                            | {pg_monitor}
 operator       | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 relation_id_22 |                                                            | {}
 replication    | Replication                                                | {}
 rewind         |                                                            | {}
```

Look for the role that matches the format `relation_id_xx` (for example, `relation_id_22`).

Next, run the following query:

```psql
SELECT format('ALTER TABLE %I OWNER TO "relation_id_22";', tablename) FROM pg_tables WHERE schemaname = 'public' \gexec
```

This SQL switches the owner of all tables in the public schema from `operator` to `relation_id_22`.

The database has now been successfully migrated from the machine charm PostgreSQL to the K8s charm PostgreSQL, and can now be used with the Livepatch Server K8s charm.

## Migrate patches

The procedure for migrating patches depends on the patch storage configuration set for the machine charm.

> Note: The filesystem storage option is not recommended as it prevents running multiple Livepatch Server pods, the filesystem cannot be shared across pods, and will be lost on pod restart. Instead, use PostgreSQL or swift/s3 buckets and resync the patch storage.

### PostgreSQL

For migrating the database, follow the same steps provided in the Database migration section, but with the patch storage database.

Once the database is migrated, update the charm config to use PostgreSQL for patch storage and provide the connection string:

```bash
juju config canonical-livepatch-k8s patch-storage.type=postgres
juju config canonical-livepatch-k8s patch-storage.postgres-connection-string=<postgres-connection-string>
```

If the same database is being used for patch storage as for the main Livepatch Server database, only the `patch-storage.type` key needs to be set; Juju handles the connection string via the `livepatch-postgres` integration.

After migrating the database, ensure access to the patches by querying for available patches via the admin tool:

```bash
canonical-livepatch-server-admin.livepatch-admin storage patches
```

If the migration was successful, a list of patch payloads for each patch version will be displayed.

### Object storage

The `swift` and `s3` patch storage types imply that the patches are stored in a remote AWS S3 or Swift bucket. This means that the migration of the configuration values and database migration from the reactive charm to the K8s charm is sufficient. Only the network connectivity between the Livepatch K8s charm units and the remote bucket should be verified. This might require modification of firewall rules depending on the setup.

When using object storage, such as AWS S3 or a Swift bucket, relevant connection parameters have been copied over with the configuration migration. The only required action is to refresh the patch storage:

```bash
canonical-livepatch-server-admin.livepatch-admin storage refresh
```

This adds patches found in storage to the distribution records in the Livepatch database.