---
myst:
  html_meta:
    description: "Remove a Livepatch Server snap or charm deployment and clean up its databases, patch storage, credentials, snapshots, and retained copies."
---


(server-how-to-guides-decommission-livepatch-server-securely)=

# Decommission a Livepatch Server securely

The following procedures describe how to securely remove the Livepatch Server and all associated data from the environment.

## Identify resources before removal

Database and storage deletion is destructive, so identify the exact resources belonging to the deployment first and keep only the recovery copies required by your retention policy. Record the application name, units or hosts, PostgreSQL database and role names, patch-storage backend and location, Juju storage identifiers, referenced secrets, and storage credentials; do not infer database or role names from the Juju application name. Inventory configuration exports, administrator login records, snapshots, database backups, object versions, and external logs; see {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`. Stop scheduled administrative jobs, synchronization automation, and offline uploads, and remove the ingress or reverse-proxy route and unneeded DNS and firewall entries.

## Remove a standalone snap

This procedure is not for the snap managed by the machine charm. Stop the service and remove the snap with `--purge`; without it, snap removal creates a snapshot retained for 31 days. Purging does not remove earlier snapshots, so list them and forget each one your retention policy permits you to remove, using its identifier from the `Set` column:

```shell
sudo snap stop canonical-livepatch-server
sudo snap remove --purge canonical-livepatch-server
sudo snap saved canonical-livepatch-server
sudo snap forget '<set-id>' canonical-livepatch-server
```

## Remove a Juju-managed deployment

Use this procedure for both the machine charm and Kubernetes charm. Application removal does not clean up PostgreSQL, separately configured patch storage, or external copies, so continue with the following sections rather than destroying the whole model. The removal command also supports `--destroy-storage`; use it only after identifying the affected storage and retained recovery copies, and do not treat it as a guarantee that every Kubernetes volume, cloud disk, or snapshot has been erased. Inspect the model and attached storage, then remove the actual Livepatch application, and repeat the inspection after removal completes, including retained machines and Kubernetes volumes:

```shell
juju status --relations
juju storage
juju remove-application '<livepatch-application>'
```

## Remove PostgreSQL data

Have the database administrator identify the application database, any separate PostgreSQL patch database, and their dedicated roles, and confirm that application sessions have ended. For a charm-managed PostgreSQL service, use its supported removal procedure and check what relation removal has already deleted. Dropping the application database removes its registrations, reports, authentication data, and patch metadata, and patch binaries if PostgreSQL patch storage uses it. If deletion fails because sessions or dependencies remain, resolve them with the database administrator rather than broadening deletion. Where direct SQL administration is permitted, connect to a maintenance database and drop each confirmed database, then each dedicated role that still exists and is no longer needed:

```sql
DROP DATABASE "<database-name>";
DROP ROLE "<dedicated-database-role>";
```

## Remove patch-storage data

Use the location recorded before removal and the storage provider's supported deletion procedures:

| Backend | Cleanup |
| ------- | ------- |
| Filesystem | Remove the configured patch directory from every retained host or volume serving patches. The standalone snap default is `/var/snap/canonical-livepatch-server/common/patches`; check for a custom `patch-storage.filesystem-path`. |
| PostgreSQL | Remove the database containing patch data as described above. |
| Amazon S3 | Remove the patch objects and any unneeded bucket, including noncurrent versions, delete markers, and replicas. |
| OpenStack Swift | Remove the patch objects and any unneeded container, including versioned and backup copies. |
| Google Cloud Storage | Remove the patch objects and any unneeded bucket, accounting for object versions, soft-deleted objects, and retention policies. |
| Azure Blob Storage | Remove the patch blobs and any unneeded container, accounting for blob versions, snapshots, soft deletion, and retention policies. |
| IBM Cloud Object Storage or Oracle Cloud Infrastructure Object Storage | Remove the patch objects and any unneeded bucket, accounting for object versions and retention policies. |

## Remove configuration secrets

Remove deployment-specific Juju secrets and grants, storage access credentials, environment files, automation secrets, and temporary configuration exports that are no longer required, including those held outside the Juju model or snap host. On each administrator workstation, remove saved login records for this server rather than deleting the whole administrator token store. Replace `<server-ip>` with the IP address or hostname of the retired server's frontend, and use the same scheme as when you logged in. Repeat for other addresses used to access the same deployment, and run the command as each user who logged in, omitting `sudo` for records created without it:

```shell
sudo livepatch-admin --url 'http://<server-ip>' logout
```

## Handle logs, backups, and remaining copies

Apply the retention policy to local and centralized logs, exported reports, database archives, volume and snap snapshots, object versions, and configuration backups. Snap application logs are held in the host's systemd journal; manage their expiry through the journal's retention policy rather than deleting the entire journal. For Kubernetes, account for cluster log retention and external collectors; see {ref}`logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`. Record which resources were deleted and which copies remain, with their retention period and owner. Application removal and logical deletion do not guarantee physical erasure from underlying media or provider replicas; where secure erasure is required, use the infrastructure or storage provider's supported media-sanitization procedures.
