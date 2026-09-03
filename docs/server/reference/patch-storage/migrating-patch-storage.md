---
myst:
  html_meta:
    description: "Migrate patches between storage backends - learn about this topic in Livepatch on-prem."
---

(server-reference-migrating-patch-storage)=

# Migrating patches between storage backends

This guide covers migrating between the file-based patch storage backends (filesystem, S3, GCS, Azure, IBM, Oracle, Swift). Each of these stores patches as a flat set of files, each keyed by its filename, so the same set of patch files works unmodified on any of them: migrating between backends is simply a matter of copying the patch files across, then updating [`patch-storage.type`](/server/reference/platform/configuration.md) and its associated options to point at the new backend.

The Postgres backend (`patch-storage.postgres-connection-string`) is not covered here, since it stores patches as rows in a database rather than as files, and cannot be migrated with `rsync` in the same way.

 The recommended tool for copying patches is `rsync`. If you need checksum-based verification, add `--checksum` (note that this can be slower). Disable patch synchronisation in the config first (so no new patches arrive, and avoid running any manual syncs), then run `rsync` once to copy the existing patches across.

## Filesystem to filesystem

If either the source or destination (or both) is the filesystem backend, `rsync` can be used directly, locally or over SSH:

```bash
rsync -avz --progress /var/snap/canonical-livepatch-server/common/patches/ user@remote-host:/path/to/new/patches/
```

## Migrating to or from object storage

For the object storage backends (S3, GCS, Azure, IBM, Oracle, Swift), mount the bucket or container as a local directory with [rclone](https://rclone.org/), which supports all of these backends (including any S3-compatible endpoint, such as IBM COS or MinIO), then `rsync` into or out of the mount as if it were a normal directory.

1. Disable [`patch-sync`](/server/reference/patch-management/patch-sync-filters.md) in the config so no new patches are synced to the on-prem server, and avoid running any manual syncs, while the migration is in progress.
2. Install rclone and configure a remote for the bucket/container, following [rclone's documentation](https://rclone.org/docs/) for the relevant backend (`s3`, `google cloud storage`, `azureblob`, `swift`, or `oracle-object-storage`).
3. Mount the remote:
   ```bash
   mkdir -p /mnt/livepatch-patches
   rclone mount <remote-name>:<bucket-or-container> /mnt/livepatch-patches --daemon
   ```
4. Run `rsync` to copy the existing patches across:
   ```bash
   rsync -avz --progress /var/snap/canonical-livepatch-server/common/patches/ /mnt/livepatch-patches/
   ```
5. Verify the file counts match between source and destination, then update the [patch storage config](/server/reference/platform/configuration.md) to the new backend and re-enable patch-sync.

The same approach applies in reverse (object storage to filesystem, or between two object storage backends) by mounting both sides and running `rsync` between the two mount points.
