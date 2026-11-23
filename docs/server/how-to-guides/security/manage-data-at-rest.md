---
myst:
  html_meta:
    description: "Protect Livepatch Server data at rest, export and restore PostgreSQL backups securely, and coordinate patch-storage migration and recovery."
---

(server-how-to-guides-manage-data-at-rest)=

# Manage data at rest

Use this guide to protect, back up, and restore data belonging to a machine-charm, Kubernetes-charm, or standalone-snap deployment. It covers PostgreSQL backup and restore, and links to procedures for copying filesystem and object-storage patches. It does not cover converting PostgreSQL patch storage to or from filesystem or object storage.

The Juju examples use the application name `livepatch`. Substitute your application's name and the actual database, storage, and account details throughout.

## Identify the data to protect

Record the resources belonging to this deployment before exporting or moving data:

| Data | Location and backup scope |
| ---- | ------------------------- |
| Client registrations, authentication tokens, patch metadata, stored machine reports, and administrator session keys | The application PostgreSQL database. Back up the complete database, not just report or patch tables. |
| Patch binaries | The configured filesystem, PostgreSQL, Amazon S3, OpenStack Swift, Google Cloud Storage, or Azure Blob Storage backend. A separate PostgreSQL patch database needs its own backup. |
| Deployment configuration and credentials | Juju configuration, referenced secrets, or standalone snap configuration and associated credential files. Configuration exports do not include the contents of referenced Juju secrets. |
| Logs, report exports, snapshots, and existing backups | Local or external destinations managed separately from the application database. See {ref}`logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>` for log retention. |

Record the deployed server revision, database version, storage settings, required extensions, and the identities that need access after recovery. Keep encryption keys and the means to recover credentials in an approved credential-management system, separate from the encrypted backup.

## Configure storage protection

Enable encryption at rest through the host or volume for filesystem storage, through your PostgreSQL deployment's storage protection, or through the object-storage provider. Include replicas, snapshots, exports, and backup destinations in the policy.

Restrict database, directory, bucket, and container access to the workload and authorized operators. Separate backup administration from ordinary client access. Protect storage encryption keys and ensure that the recovery environment can obtain them. Follow the backend-specific controls in {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`.

Prepare a new, restricted directory on an encrypted administration volume before creating exports:

```shell
umask 077
mkdir livepatch-backup
cd livepatch-backup
```

The following export examples run from this directory. Keep it outside version control and public or shared download locations. A PostgreSQL archive is not encrypted merely because it uses the custom archive format.

For either charm, export the application configuration:

```shell
juju config livepatch > charm-config.yaml
```

For a standalone snap instead:

```shell
sudo snap get canonical-livepatch-server -d lp > snap-config.json
```

Treat these files as sensitive. Preserve referenced secrets and credential files through your approved recovery process; a secret URI alone is not a recoverable credential. Do not put database passwords, storage keys, or access tokens in command arguments or transfer URLs.

## Establish a consistent recovery point

Schedule a maintenance window when taking a coordinated database and patch-store backup or performing a migration. Drain client and administrator traffic, pause scheduled administrative jobs, and prevent old and new deployments from writing during the operation.

Disable scheduled patch synchronization through the supported configuration interface. Record its previous value so that an air-gapped or intentionally disabled deployment is not enabled accidentally.

For either charm:

```shell
juju config livepatch patch-sync.enabled=false
```

For a standalone snap:

```shell
sudo snap set canonical-livepatch-server lp.patch-sync.enabled=false
```

Disabling synchronization is not a complete write freeze. Finish or stop outstanding synchronization, manual imports, patch uploads, and other administrative writes. Account for direct object-storage uploads as well as requests passing through the server.

Use your deployment's maintenance procedure to stop application writers or isolate their database and storage access. For charms, keep changes under Juju and provider control rather than changing the managed snap or container directly. Removing ingress alone does not stop background tasks or direct access to units.

Verify that writers are stopped or denied access before the final backup or copy. Keep that restriction in place through cutover. A PostgreSQL dump provides a consistent view of one database, but separate database and object-store backups do not automatically share a recovery point.

For a machine-charm migration, the [migration guide](/server/how-to-guides/deployment/migrate-to-the-new-machine-charm.md) includes deployment-specific writer-disconnection and database-ownership checks. Do not remove database relations merely to take a routine backup.

## Export PostgreSQL data

Use the PostgreSQL provider's supported backup procedure, or take a complete logical backup with `pg_dump`. Use client tools matching the source database's major version and confirm compatibility with the destination. Arrange any cross-version database upgrade with the database administrator.

Run database tools from a protected administration host over a trusted network, or use TLS configured to validate the database certificate and hostname. The commands below do not configure transport protection. Obtain credentials through the database provider's documented interface and use the password prompt or an approved protected credential file.

With an account authorized to read the complete source database, including large objects:

```shell
pg_dump --host='<source-database-host>' --username='<backup-user>' \
  --dbname='<database-name>' --password --format=custom -b \
  --file=livepatch.dump
pg_restore --list livepatch.dump
sha256sum livepatch.dump > livepatch.dump.sha256
```

Check that each command succeeds before continuing. Archive listing checks readability; it does not prove that the database can be restored.

For PostgreSQL patch storage, retain large objects as well as the database records that refer to them. Do not use a table-only or schema-only export as a patch backup. If patches use the application database, one complete dump covers both. Otherwise, repeat the backup for the separate patch database using distinct archive and checksum filenames while writers remain stopped.

A database dump does not preserve deployment configuration, external patch files, or cluster-wide roles. Record how the destination provider will supply the required roles, privileges, and extensions.

## Transfer backups and patches

Transfer database archives and configuration through an authenticated, encrypted channel such as SSH or a protected backup service. Verify the destination host or service identity; do not disable certificate or SSH host-key checks. Keep destination access restricted and enable encryption at rest before uploading data.

Retain checksum records in a protected location and compare the received archive against the trusted source record. A checksum stored beside an archive does not prevent an attacker from replacing both.

For filesystem and object-storage patches, follow {ref}`Migrate patches between storage backends <server-reference-migrating-patch-storage>`. Preserve patch filenames and contents and take a matching database backup.

## Restore and check the destination

Keep the destination isolated from production traffic and application writers until restoration and ownership checks are complete. Connecting a workload to the destination can start writes or automatic schema migrations.

Use your database provider's recovery procedure for provider-managed backups. For a logical archive, have the database administrator prepare a new, empty destination database, the required extensions, and the intended workload role. Do not restore over a live or shared database.

Before restoring, verify the received archive against the protected checksum record. For the filenames above, run from the directory containing the archive:

```shell
sha256sum --check livepatch.dump.sha256
```

When moving to a different workload role, the following example restores objects under the destination role instead of recreating source ownership and grants:

```shell
pg_restore --host='<destination-database-host>' \
  --username='<restore-administrator>' --dbname='<empty-destination-database>' \
  --password --role='<destination-workload-role>' \
  --no-owner --no-privileges --exit-on-error --single-transaction livepatch.dump
```

The restore administrator must be allowed to assume the destination role, and that role must have the privileges needed to create the application objects. Discover charm-managed role names from the destination relation; do not infer them from the Juju application name. Have the database administrator check ownership and required grants, including access to PostgreSQL patch large objects, before starting the workload. Do not ignore restore errors.

Restore any separate patch database and the matching patch-store backup. Reapply reviewed configuration and credential references through Juju or the standalone snap interface. Ensure that filesystem paths are accessible to the workload and that storage permissions and encryption settings remain correct.

Allow only the destination deployment to resume writes. Verify administrator authentication, existing client access, tier assignments, and patch downloads. If machine reporting is enabled, verify report storage too. Check application logs and database and storage errors; a healthy process or an HTTP 200 response alone is not sufficient.

## Cut over and retain recovery copies

Switch the frontend only after destination checks pass, preserving TLS, routing restrictions, and any separate patch-download endpoint. Restore synchronization to its recorded setting and resume scheduled jobs deliberately. Keep old writers disconnected.

Retain the original database and patch-store recovery point until the migration is accepted. If recovery is needed, isolate the failed destination and restore a matching database, patch store, configuration, and compatible server version. Do not reconnect an older workload to a database that a newer version has migrated. Writes made after the backup are not recovered by restoring it.

Apply your retention policy to temporary exports, transfer-host copies, backup versions, snapshots, and configuration archives. Remove temporary copy credentials and unneeded local files through your approved process without deleting the retained recovery copy. Local report cleanup does not delete exported data or older backups. Removing a file is not a guarantee of physical erasure from underlying media or storage-provider replicas; use the relevant storage provider's deletion and media-sanitization procedures.
