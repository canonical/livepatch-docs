---
myst:
  html_meta:
    description: "Remove a Livepatch Server snap or charm deployment and clean up its databases, patch storage, credentials, snapshots, and retained copies."
---


(server-how-to-guides-decommission-livepatch-server-securely)=

# Decommission a Livepatch Server securely

Use this guide to remove a standalone snap, machine-charm, or Kubernetes-charm deployment and clean up its data and credentials according to your retention policy.

```{warning}
Database and storage deletion is destructive. Identify the exact resources belonging to the deployment before removing them. Keep only the recovery copies required by your retention policy, and protect those copies separately.
```

## Identify resources before removal

Record the application name, units or hosts, PostgreSQL database and role names, patch-storage backend and location, Juju storage identifiers, and referenced secrets. Include any separate PostgreSQL patch database, custom filesystem paths, and storage credentials. Do not infer database or role names from the Juju application name.

Inventory configuration exports, administrator login records, snap snapshots, cloud or volume snapshots, database backups, object versions, and external logs. For backup and retention procedures, see {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`.

Stop scheduled administrative jobs, synchronization automation, and offline uploads. Restrict frontend and direct API access while removing the deployment. Remove its ingress or reverse-proxy route and unneeded DNS and firewall entries through the relevant provider's procedures.

## Remove a standalone snap

This procedure is not for the snap managed by the machine charm. Use the Juju procedure below for charm-managed deployments.

Stop the service, then remove the snap without creating an automatic removal snapshot:

```shell
sudo snap stop canonical-livepatch-server
sudo snap remove --purge canonical-livepatch-server
```

Without `--purge`, snap removal normally creates a snapshot retained for 31 days. The purge option does not remove previously created snapshots.

List existing snapshots:

```shell
sudo snap saved canonical-livepatch-server
```

For each snapshot that your retention policy permits you to remove, use its actual identifier from the `Set` column:

```shell
sudo snap forget '<set-id>' canonical-livepatch-server
```

Continue with database, patch-storage, credential, and external-copy cleanup below.

## Remove a Juju-managed deployment

Use this procedure for both the machine charm and Kubernetes charm. Inspect the model and attached storage before removing the application:

```shell
juju status --relations
juju storage
```

After recording the resources to clean up, remove the actual Livepatch application:

```shell
juju remove-application '<livepatch-application>'
```

Wait for removal to complete and inspect the model and storage again. If you also intend to destroy storage attached to the application, the removal command supports `--destroy-storage`; use it only after identifying the affected storage and retained recovery copies.

Do not treat this flag as a guarantee that every Kubernetes volume, cloud disk, snapshot, or external data service has been erased. Check the storage provider's retention and reclaim behavior. Inspect retained machines for workload data and snap snapshots, and retained Kubernetes volumes for application data. Use the relevant Juju, Kubernetes, or infrastructure-provider procedure for each identified resource.

Application removal does not by itself complete cleanup of PostgreSQL, separately configured patch storage, or external copies. Continue with the following sections rather than destroying the whole model as a shortcut.

## Remove PostgreSQL data

Have the database administrator identify the application database, any separate PostgreSQL patch database, and their dedicated access roles. Confirm that application sessions have ended before deletion.

For a charm-managed PostgreSQL service, use its supported database and role removal procedure. Check what relation removal has already deleted before issuing further commands.

Where your database provider permits direct SQL administration, connect to a maintenance database rather than the database being removed. For each confirmed database that still exists:

```sql
DROP DATABASE "<database-name>";
```

Dropping the application database removes its stored registrations, reports, authentication data, and patch metadata. It also removes patch binaries if PostgreSQL patch storage uses that database. Remove a separate PostgreSQL patch database explicitly.

After checking that a dedicated database role still exists and is no longer needed, remove it through the provider or, where supported:

```sql
DROP ROLE "<dedicated-database-role>";
```

If deletion fails because sessions or dependencies remain, resolve them through the database administrator. Do not ignore the failure or broaden deletion to unrelated databases or roles.

## Remove patch-storage data

Use the location recorded before removal:

| Backend | Cleanup |
| ------- | ------- |
| Filesystem | Remove the configured patch directory from every retained host or volume serving patches. The standalone snap default is `/var/snap/canonical-livepatch-server/common/patches`; check for a custom `patch-storage.filesystem-path`. Snap removal can already have removed data within its own directories. |
| PostgreSQL | Remove the database containing patch data as described above, including a separate patch database if configured. |
| Amazon S3 | Remove the deployment's patch objects and any bucket no longer required. Account for noncurrent versions, delete markers, and retained replicas or backups. |
| OpenStack Swift | Remove the deployment's patch objects and any container no longer required. Check configured versioning and backup locations. |
| Google Cloud Storage | Remove the deployment's patch objects and any bucket no longer required. Account for object versions, soft-deleted objects, and retention policies. |
| Azure Blob Storage | Remove the deployment's patch blobs and any container no longer required. Account for blob versions, snapshots, soft deletion, and retention policies. |

Use the storage provider's supported deletion procedures. Do not assume that deleting current objects removes historical versions, snapshots, or backup copies.

## Remove credentials and configuration copies

Remove deployment-specific Juju secrets and grants, storage access credentials, environment files, automation secrets, and temporary configuration exports that are no longer required. Include credentials held outside the Juju model or snap host.

On each administrator workstation, remove saved login records for this server:

```shell
sudo livepatch-admin --url '<hosted url>' logout
```

Replace `<hosted url>` with the retired server's complete frontend URL, including its scheme. Repeat for other URLs used to access the same deployment. Run the command as each user who logged in, omitting `sudo` for records created without it.

The logout command targets the selected server's saved login. Do not delete the whole administrator token store to clean up one deployment.

## Handle logs, backups, and remaining copies

Apply the retention policy to local and centralized logs, exported reports, database archives, volume and snap snapshots, object versions, and configuration backups. Retained copies can contain credentials and fleet information.

Snap application logs are held in the host's systemd journal. Manage their expiry through the journal's retention policy rather than deleting the entire host journal to remove one service's records. For Kubernetes, account for cluster log retention as well as external collectors. See {ref}`logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.

Record which resources were deleted and which copies remain, together with their retention period and owner. Confirm that the application is gone and that no unintended database, patch-storage, credential, or snapshot resources remain.

Application removal and logical deletion do not guarantee physical erasure from underlying media or provider replicas. Where sanitization is required, use the infrastructure or storage provider's supported media-sanitization procedures.