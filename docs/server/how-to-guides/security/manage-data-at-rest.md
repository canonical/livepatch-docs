---
myst:
  html_meta:
    description: "Protect Livepatch Server data at rest, export and restore PostgreSQL backups securely, and coordinate patch-storage migration and recovery."
---

(server-how-to-guides-manage-data-at-rest)=

# Manage data at rest

Use this guide to protect, back up, and restore data belonging to a machine-charm, Kubernetes-charm, or standalone-snap deployment. The Juju examples use the application name `livepatch`; substitute your actual application, database, storage, and account details.

## Identify and protect the data

Livepatch Server stores client registrations, authentication tokens, patch metadata, stored machine reports, and administrator session keys in its PostgreSQL database. Patch binaries are held in the configured {ref}`patch storage backend <server-reference-patch-storage>`. Deployment configuration and credentials are held in Juju configuration and secrets, or in the standalone snap configuration and credential files. Logs, report exports, and snapshots are managed separately; see {ref}`logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.

Enable encryption at rest for each location through the host or volume, your PostgreSQL deployment, or the object-storage provider, including snapshots and backup destinations. Restrict access to the workload and authorized operators, and keep encryption keys separate from encrypted backups; see {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`.

## Back up the data

Schedule a maintenance window. Disable patch synchronization with `juju config livepatch patch-sync.enabled=false`, or `sudo snap set canonical-livepatch-server lp.patch-sync.enabled=false` for the standalone snap, and record the previous value. Disabling synchronization is not a complete write freeze: also stop manual imports, patch uploads, and other application writers, and keep them stopped until the final backup or copy is complete. For a machine-charm migration, see the [migration guide](/server/how-to-guides/deployment/migrate-to-the-new-machine-charm.md).

From a restricted directory outside version control, export the configuration with `juju config livepatch > charm-config.yaml`, or `sudo snap get canonical-livepatch-server -d lp > snap-config.json` for the standalone snap. These exports do not include the contents of referenced Juju secrets. Then use your PostgreSQL provider's backup procedure, or take a complete logical backup, including large objects, with `pg_dump` client tools matching the source database's major version:

```shell
pg_dump --host='<source-database-host>' --username='<backup-user>' --dbname='<database-name>' \
  --password --format=custom -b --file=livepatch.dump
pg_restore --list livepatch.dump
sha256sum livepatch.dump > livepatch.dump.sha256
```

If PostgreSQL patch storage uses a separate database, back it up the same way with distinct filenames. For filesystem and object-storage patches, follow {ref}`Migrate patches between storage backends <server-reference-migrating-patch-storage>`. Transfer archives over an authenticated, encrypted channel, and keep the checksum record apart from the archive.

## Restore and verify

Have the database administrator prepare a new, empty destination database and workload role, isolated from production writers; connecting a workload can start writes or schema migrations. Verify the archive, then restore objects under the destination role:

```shell
sha256sum --check livepatch.dump.sha256
pg_restore --host='<destination-database-host>' --username='<restore-administrator>' \
  --dbname='<empty-destination-database>' --password --role='<destination-workload-role>' \
  --no-owner --no-privileges --exit-on-error --single-transaction livepatch.dump
```

Restore any separate patch database and the matching patch store, and reapply configuration through Juju or the snap. Verify administrator authentication, client access, tier assignments, and patch downloads before allowing only the destination to resume writes, then restore synchronization to its recorded setting. Retain the original database and patch-store recovery point until the migration is accepted. Restoring a backup does not recover writes made after it, and an older workload must not reconnect to a database that a newer version has migrated.
