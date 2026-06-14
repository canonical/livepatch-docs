---
myst:
  html_meta:
    description: "How to securely decommission the client with Livepatch client."
---


(client-how-to-guides-how-to-securely-decommission-the-livepatch-client)=

# How to securely decommission the Livepatch client

The canonical-livepatch snap holds user, application and configuration data during its lifecycle. The canonical-livepatch snap also produces application and security logs. This guide provides an overview on how to decommission the Livepatch client securely.

## Decommission the snap data

Decommissioning the snap data requires deleting the snap and any remnant data that the snap has created. The following steps highlight how this can be done:

1. Remove and delete the snap. Running this command will unmount the snap and remove all snap data under `/var/snap/canonical-livepatch/` and `/home/<username>/snap/canonical-livepatch/`. However, **a copy of the snap data is retained as a snapshot for 30 days**. This snapshot can be restored or manually retrieved.

```
sudo snap remove canonical-livepatch
```

2. To remove and **delete a snap without creating any new data snapshots**, use the `--purge` flag with the `snap remove` command. While this will remove the snap data and not create any new snapshots, previously created snapshots will not be deleted.

```
sudo snap remove --purge canonical-livepatch
```

3. Any remnant data snapshots of the canonical-livepatch snap can be found and deleted in a two-step process.

- The first step involves finding the snapshots.

```
sudo snap saved canonical-livepatch
```

- If any snapshots exist for the canonical-livepatch snap, running the command will display the snapshot IDs under the Set column. The snapshot IDs can then be used to delete the snap data snapshots.
  For example, if there are 3 existing snapshots for the canonical-livepatch snap with IDs 1, 2 and 3, run the following command to delete the snapshots

```
printf "%s\n" 1 2 3 | xargs -I {} sudo snap forget {} canonical-livepatch
```

## Decommission the snap logs

The canonical-livepatch snap generates two kinds of logs during its lifecycle. These include the security logs and the application logs. The security logs are always stored in the logs file under the [$SNAP_COMMON](https://snapcraft.io/docs/reference/administration/data-locations/#system-data) directory for the snap. Therefore, these logs can be securely deleted and purged by deleting the snap and all of its snapshots, using the steps mentioned above.

The application logs on the other hand, are captured by the systemd-journald service. Use `journalctl --directory=/var/log/journal -u snap.canonical-livepatch.canonical-livepatchd.service` to check if logs for the canonical-livepatch snap exist in the journal. These logs are interleaved with other system logs in the journal, and therefore, cannot be removed without impacting other system logs. The remnant logs from the snap will be removed when the journal automatically rotates and vacuums the logs as per the system configurations for the journald service. See the manpage for the [journald service](https://man7.org/linux/man-pages/man8/systemd-journald.service.8.html) and [journald configuration](https://man7.org/linux/man-pages/man5/journald.conf.5.html) for more information.
