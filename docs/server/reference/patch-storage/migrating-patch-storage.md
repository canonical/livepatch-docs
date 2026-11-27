---
myst:
  html_meta:
    description: "Migrate patches between storage backends - learn about this topic in Livepatch on-prem."
---

(server-reference-migrating-patch-storage)=

# Migrating patches between storage backends

This guide covers migrating between the file-based patch storage backends (filesystem, S3, GCS, Azure, IBM, Oracle, Swift). Each of these stores patches as a flat set of files, each keyed by its filename, so the same set of patch files works unmodified on any of them: migrating between backends is simply a matter of copying the patch files across, then updating [`patch-storage.type`](/server/reference/platform/configuration.md) and its associated options to point at the new backend.

The Postgres backend (`patch-storage.postgres-connection-string`) is not covered here, since it stores patches as rows in a database rather than as files, and cannot be migrated with `rsync` in the same way.

Preserve every patch's filename, without an added directory prefix. For PostgreSQL backup and restore, see {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`.

Before copying, record the current [patch-storage configuration](/server/reference/platform/configuration.md#patch-storage), back up the application database and source patches, and prepare a dedicated destination with encryption at rest and restricted access. Disable `patch-sync.enabled` through Juju, or `lp.patch-sync.enabled` for the standalone snap, and stop manual synchronization, imports, and uploads, including direct uploads to object storage, until the migration is complete. Use authenticated, encrypted connections, and do not delete source objects or use a copy mode that removes destination data.

## Filesystem to filesystem

Use `rsync`, over SSH for a remote destination. The source path below is the standalone snap default; check `patch-storage.filesystem-path` for your deployment. The trailing slash on the source copies its contents rather than adding a `patches` subdirectory. After copying, compare filenames, sizes, and content checksums between source and destination, and set ownership and permissions for the destination workload. With multiple units using separate filesystem stores, copy and verify the patches on every unit serving downloads.

```bash
rsync --archive --no-owner --no-group --checksum --progress --rsh=ssh \
  /var/snap/canonical-livepatch-server/common/patches/ '<copy-user>@<destination-host>:/protected/patches/'
```

## Migrating to or from object storage

Use [rclone](https://rclone.org/) with a named remote for each object store, configured following [rclone's documentation](https://rclone.org/docs/). A copy does not preserve provider permissions, encryption-key settings, or lifecycle policies; configure them on the destination separately. For object storage to filesystem or between object stores, use `<source-remote>:<source-bucket-or-container>` as the source. For example, copy a filesystem patch store to object storage, then verify the contents by downloading them, which can incur retrieval charges:

```bash
rclone copy /var/snap/canonical-livepatch-server/common/patches/ \
  '<destination-remote>:<destination-bucket-or-container>' --progress
rclone check /var/snap/canonical-livepatch-server/common/patches/ \
  '<destination-remote>:<destination-bucket-or-container>' --download
```

## Switch storage and verify access

After verification, update `patch-storage.type` and the destination backend's options through [Juju or standalone snap configuration](/server/reference/platform/configuration.md#setting-configuration). Verify patch downloads from the new store, including from the client network, then resume synchronization only if it was previously enabled. Keep the source unchanged until the migration is accepted.
