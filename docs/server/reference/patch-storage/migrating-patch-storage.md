---
myst:
  html_meta:
    description: "Migrate patches between storage backends - learn about this topic in Livepatch on-prem."
---

 
(server-reference-migrating-patch-storage)=

# Migrating patches between storage backends

Every patch storage backend supported by Livepatch Server (filesystem, S3, GCS, Azure, IBM, Oracle, Swift) stores patches as a flat set of files, each keyed by its filename. This means the same set of patch files works unmodified on any backend: migrating between backends is simply a matter of copying the patch files across, then updating [`patch-storage.type`](/server/reference/platform/configuration.md) and its associated options to point at the new backend.

The recommended tool for copying patches is `rsync`, since it performs a checksum-verified copy and, on repeated runs, only transfers files that have changed. This allows for a near zero-downtime migration: run an initial sync while the current backend is still serving traffic, then run a final, much shorter sync during a brief maintenance window to catch any patches that arrived in between.

## Filesystem to filesystem

If either the source or destination (or both) is the filesystem backend, `rsync` can be used directly, locally or over SSH:

```bash
rsync -avz --progress /var/snap/canonical-livepatch-server/common/patches/ user@remote-host:/path/to/new/patches/
```

## Migrating to or from object storage

For the object storage backends (S3, GCS, Azure, IBM, Oracle), mount the bucket or container as a local directory with [rclone](https://rclone.org/), which supports all of these backends (including any S3-compatible endpoint, such as IBM COS or MinIO), then `rsync` into or out of the mount as if it were a normal directory.

1. Install rclone and configure a remote for the bucket/container, following [rclone's documentation](https://rclone.org/docs/) for the relevant backend (`s3`, `google cloud storage`, `azureblob`, or `oracle-object-storage`).
2. Mount the remote:
   ```bash
   mkdir -p /mnt/livepatch-patches
   rclone mount <remote-name>:<bucket-or-container> /mnt/livepatch-patches --daemon
   ```
3. Run an initial sync while the current backend is still in use:
   ```bash
   rsync -avz --progress /var/snap/canonical-livepatch-server/common/patches/ /mnt/livepatch-patches/
   ```
4. Stop the Livepatch server (or disable [`patch-sync`](/server/reference/patch-management/patch-sync-filters.md) so no new patches are written) and run a final sync to catch any changes from step 3:
   ```bash
   rsync -avz --progress --delete /var/snap/canonical-livepatch-server/common/patches/ /mnt/livepatch-patches/
   ```
5. Verify the file counts match between source and destination, then update the [patch storage config](/server/reference/platform/configuration.md) to the new backend and restart the server.

The same approach applies in reverse (object storage to filesystem, or between two object storage backends) by mounting both sides and running `rsync` between the two mount points.
