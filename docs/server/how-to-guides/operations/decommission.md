---
myst:
  html_meta:
    description: "How to decommission with Livepatch on-prem."
---


(server-how-to-guides-decommission-livepatch-server-securely)=

# Decommission Livepatch server securely

This document provides guidance on securely removing the Livepatch Server and all associated data from the environment.

## Decommission a snap deployment

### Remove the snap

1. Stop and remove the snap. This removes the snap and its data, but a snapshot is retained for 31 days:

```shell
sudo snap remove canonical-livepatch-server
```

2. To remove the snap without creating a snapshot, use the `--purge` flag:

```shell
sudo snap remove --purge canonical-livepatch-server
```

3. Check for and remove any previously created snapshots:

```shell
sudo snap saved canonical-livepatch-server
```

If snapshots exist, delete them using the snapshot IDs shown in the `Set` column:

```shell
printf "%s\n" 1 2 3 | xargs -I {} sudo snap forget {} canonical-livepatch-server
```

### Remove the database

Drop the Livepatch database from PostgreSQL. Connect to PostgreSQL as a superuser and run:

```sql
DROP DATABASE livepatch;
DROP USER livepatch;
```

### Remove application logs

Snap application logs are captured by systemd-journald and cannot be removed independently without impacting other system logs. These logs will be automatically removed when the journal rotates according to the system's journald configuration. See the [journald configuration manpage](https://man7.org/linux/man-pages/man5/journald.conf.5.html) for details on configuring rotation and retention.

### Remove patch storage data

Remove patch files from the configured storage backend:

- **Filesystem**: Delete the patches directory (default: `/var/lib/livepatch/patches`).
- **S3**: Delete the configured S3 bucket or its contents.
- **Swift**: Delete the configured Swift container or its contents.
- **PostgreSQL**: Patch data is removed when the database is dropped.

## Decommission a charm deployment

### Remove the application

```shell
juju remove-application canonical-livepatch-server-k8s --destroy-storage
```

The `--destroy-storage` flag ensures that any persistent volumes associated with the application are deleted.

### Remove the model

To remove the entire Juju model and all associated resources:

```shell
juju destroy-model <model-name> --destroy-storage --force
```

## Remove configuration secrets

After decommissioning, ensure that any secrets stored outside the deployment are also removed:

- Juju secrets created for the application
- Environment variable files or shell history containing credentials
- Backup copies of configuration files containing admin passwords or database connection strings
- Admin tool token files stored at `~/snap/canonical-livepatch-server-admin/common/tokens` on any machine where the admin tool was used
