---
myst:
  html_meta:
    description: "How to upgrade a Livepatch on-premises deployment, perform database schema migrations, and sync patch publish timestamps."
---

(server-how-to-guides-how-to-upgrade-a-livepatch-on-prem-deployment)=

# How to upgrade a deployment

Upgrading an on-premises Livepatch Server deployment involves updating application binaries, applying database schema migrations, and optionally running a metadata-only sync to backfill patch publish timestamps.

## Step 1: Update application binaries

Depending on whether your deployment uses Juju charms or standalone snaps, follow the corresponding procedure.

### Charmed deployments (Juju)

In charmed deployments, upgrade each component application using `juju refresh`:

```bash
juju refresh livepatch
```

If you manage supporting infrastructure services via Juju, upgrade them as required:

```bash
juju refresh postgresql
juju refresh haproxy
juju refresh ubuntu-advantage
```

### Standalone snap deployments

In snap deployments, refresh the server snap:

```bash
sudo snap refresh canonical-livepatch-server
```

## Step 2: Apply database schema upgrades

When upgrading across major or minor Livepatch Server releases, new database migrations may be required.

### Charmed deployments

When a schema upgrade is required, the Livepatch application enters a blocked status visible in `juju status`. Run the `schema-upgrade` action on the leader unit:

```bash
juju run livepatch/leader schema-upgrade --wait
```

```{note}
In Juju v3, `juju run-action` was renamed to `juju run`. See the [Juju documentation on managing actions](https://documentation.ubuntu.com/juju/latest/howto/manage-actions/#run-an-action) for details.
```

If needed, restart units after the schema upgrade completes:

```bash
juju run livepatch/<desired-unit> restart --wait
```

### Snap deployments

For snap deployments, database migrations run automatically during service startup when the updated snap starts. The migration runner acquires an advisory lock on PostgreSQL to ensure that multi-unit or clustered deployments upgrade safely without race conditions.

You can also run schema migrations manually using the snap command:

```bash
sudo canonical-livepatch-server.schema-upgrade
```

Check the service logs to verify successful migration:

```bash
sudo journalctl -u snap.canonical-livepatch-server.* -n 100 --no-pager
```

### Key schema migrations

Recent Livepatch Server releases introduce database improvements that support the patch uploader and offline distribution workflows:

- **Migration 086 (`patch_file_tier.publish_time`)**: Adds `publish_time` and `sync_time` tracking to patch tiers, recording the exact point in time when each patch was made available.
- **Migration 088 (`patch_file_filename_index`)**: Adds a dedicated B-tree index on `patch_file.filename` to significantly accelerate duplicate detection and batch lookups during patch uploads.

## Step 3: Backfill publish timestamps with metadata-only sync

In earlier Livepatch Server releases, patch tiers did not record the original upstream publish timestamp from Canonical (or initialized it to the local database insertion time).

If your server has an active upstream connection (`patch-sync.enabled=true`) and you want your on-premises patch catalog to reflect accurate Canonical publish dates (for client queries, KPI reporting, and `patch list` filtering), trigger a **metadata-only sync**.

### How metadata-only sync works

Unlike a full patch synchronisation, which downloads multi-gigabyte kernel patch binaries, a metadata-only sync:

- Queries only metadata from Canonical's upstream server.
- Compares already-known local patches and updates their `publish_time` across all tiers.
- Validates local patch checksums against upstream records.
- Does **not** download binary `.tar.bz2` payloads and does **not** register newly published patches.

### Run the metadata-only sync

Trigger the metadata synchronisation job using `livepatch-admin`:

```bash
livepatch-admin sync trigger --metadata-only --wait
```

Flags:

- `--metadata-only`: Refresh publish timestamps and verify checksums of already-known patches without downloading file payloads.
- `--wait`: Block until the background synchronisation job completes (optional).
- `--timeout <duration>`: Maximum time to wait for the job (defaults to `10m`).

If you run without `--wait`, retrieve the job status using the returned job ID:

```bash
livepatch-admin job-status <job-id>
```

Or view recent sync reports:

```bash
livepatch-admin sync reports --all
```

## Step 4: Review patch sync tier configuration

If your deployment synchronises patches from an upstream server, review how tiers are mapped:

- **Full tier mirror (`patch-sync.sync-tiers=true`)**: Mirrors the entire upstream tier hierarchy locally. Patches are downloaded from upstream `stable` and mapped across all lower local tiers automatically.
- **Single-tier sync (`patch-sync.sync-tiers=false`)**:
  - `patch-sync.upstream-tier`: Specifies which tier to request from upstream (defaults to `stable`).
  - `patch-sync.ingress-tier`: Specifies the local tier into which synced patches are stored and served (defaults to your first configured tier, such as `edge` or `stable`).

Run `livepatch-admin tier list` to verify your local tiers and confirm where patches land.

