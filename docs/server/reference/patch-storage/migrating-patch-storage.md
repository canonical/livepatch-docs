---
myst:
  html_meta:
    description: "Migrate patches between storage backends - learn about this topic in Livepatch on-prem."
---

(server-reference-migrating-patch-storage)=

# Migrating patches between storage backends

Use this guide to copy patches between filesystem, Amazon S3, OpenStack Swift, Google Cloud Storage, and Azure Blob Storage backends. Preserve every patch's filename and contents; object names must remain at the location expected by the destination backend, without an added directory prefix.

PostgreSQL patch storage cannot be copied with these file-copy procedures. For PostgreSQL backup and restore, see {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`. Conversion between PostgreSQL patch storage and filesystem or object storage is outside this guide's scope.

## Prepare the migration

1. Record the current [patch-storage configuration](/server/reference/platform/configuration.md#patch-storage), synchronization setting, credentials, and any separate patch-download URL.
2. Take a protected application-database backup and a recoverable copy of the source patches. Follow {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>` for coordinated backups and write controls.
3. Prepare a dedicated destination directory, bucket, or container. Configure encryption at rest and restrict access before copying. Limit copy credentials to reading the source and writing and verifying the destination; do not grant deletion or unrelated-resource access unnecessarily.
4. Arrange maintenance and disable `patch-sync.enabled` through Juju, or `lp.patch-sync.enabled` for the standalone snap. Wait for in-progress work to finish and prevent manual synchronization, imports, uploads, and deletion. Include uploads made directly to object storage. Disabling scheduled synchronization alone does not stop every writer.
5. Keep writers blocked until the final copy and destination checks are complete. Use approved transfer tooling and authenticated, encrypted connections. Verify host or service identity rather than disabling SSH host-key or TLS certificate checks.

Do not delete source objects or use a copy mode that removes destination data automatically. If the source resource is shared, identify this deployment's patch objects explicitly rather than copying or changing unrelated data.

## Filesystem to filesystem

Use `rsync` over SSH for a remote destination. The source path below is the standalone snap default; check `patch-storage.filesystem-path` for your deployment. Prepare the destination directory with restricted access first:

```shell
rsync --archive --no-owner --no-group --checksum --progress --rsh=ssh \
  /var/snap/canonical-livepatch-server/common/patches/ \
  '<copy-user>@<destination-host>:/protected/patches/'
```

The trailing slash on the source copies its contents rather than adding a `patches` subdirectory. For a local copy, replace the SSH destination with the protected local directory.

Check the command's exit status and compare filenames, sizes, and content checksums from source and destination. `--checksum` controls which existing files are recopied; do not treat a successful transfer or matching file counts as an independent content-verification record.

Set destination ownership and permissions for the destination workload rather than preserving source account identities blindly. The confined snap must be able to access the configured path. For multiple units using separate filesystem stores, copy and verify the patches on every unit serving downloads; the charm does not replicate these files between units.

## Migrating to or from object storage

Use a provider-supported transfer tool or [rclone](https://rclone.org/) with an appropriate backend. With rclone, configure a named remote for each object store using the [backend configuration documentation](https://rclone.org/docs/). Protect its configuration file and any temporary credential material on the transfer host. Do not embed credentials in commands or disable TLS verification.

For example, copy a filesystem patch store to a configured object-storage remote:

```shell
rclone copy /var/snap/canonical-livepatch-server/common/patches/ \
  '<destination-remote>:<destination-bucket-or-container>' --progress
rclone check /var/snap/canonical-livepatch-server/common/patches/ \
  '<destination-remote>:<destination-bucket-or-container>' --download
```

Use a dedicated destination so that unrelated objects are not included in the comparison. Check that the copy succeeds before running the check, and that verification succeeds before switching storage. `--download` verifies contents by reading the data rather than relying on potentially incompatible provider checksum metadata. It adds network traffic and can incur retrieval charges.

For object storage to filesystem, reverse the source and destination. For a transfer between object stores, use `<source-remote>:<source-bucket-or-container>` as the source and the destination remote as above. Keep filenames unchanged and do not add a prefix. A data copy does not necessarily preserve provider permissions, encryption-key settings, lifecycle policies, or historical object versions; configure and verify the required destination protections separately.

## Switch storage and verify access

After content verification, update `patch-storage.type` and the destination backend's required options through [Juju or standalone snap configuration](/server/reference/platform/configuration.md#setting-configuration). Manage charm configuration through Juju rather than modifying the managed workload. Configuration changes can restart the server.

Keep production traffic restricted while checking database connectivity, patch metadata, and downloads from the new store. If clients download from a separate endpoint, update the URL and its routing, certificate trust, and storage access controls too. Verify access from the client network as well as from the server.

Resume synchronization only if it was previously enabled, and reopen traffic after the checks pass. Keep the source protected and unchanged until the migration is accepted. If rollback is needed, isolate destination writers and restore a matching database, patch store, and configuration; switching only the storage setting back can lose patches or leave metadata pointing at unavailable files.

Remove temporary transfer credentials and staging copies under your retention policy. Retain required backups and checksum records in protected storage. Deleting a source object does not necessarily remove provider versions, snapshots, or backup copies.
