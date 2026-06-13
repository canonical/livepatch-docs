---
myst:
  html_meta:
    description: "How to migrate from reactive charm to snap with Livepatch on-prem."
---


(livepatch_on_prem-how-to-guides-migration-from-livepatch-server-reactive-machine-charm-to-the-on-prem-snap)=

# Migration from Livepatch Server Reactive Machine charm to the On-prem snap

The Juju framework offered a way to write charms using the [Reactive](https://charmsreactive.readthedocs.io/en/latest/) framework and these were called [reactive charms](https://documentation.ubuntu.com/juju/3.6/reference/charm/#reactive-charm). Reactive charms have been deprecated and the Livepatch server reactive charm that allowed users to run an On-prem deployment of the Livepatch server is no longer actively maintained. The recommended way of deploying the Livepatch server currently, is to use the [Kubernetes charm](https://charmhub.io/canonical-livepatch-server-k8s) or the [Livepatch server snap package](https://snapcraft.io/canonical-livepatch-server). This document describes how to migrate a Livepatch server instance deployed with a reactive charm, to an instance deployed with the Livepatch server snap package.

Validate the details of your charmed Livepatch server deployment by connecting to your Juju controller via SSH, and running `juju status`. Confirm the charm name is "canonical-livepatch-server" and the channel the Livepatch server was installed from. The output will look like this:

|App|Status|Scale|Charm|Channel|Rev|
|------|----------|--------|----------|-------------|------|
|livepatch|active|1|canonical-livepatch-server|latest/stable|51|

View the table below to understand the charm type deployed and the status for that type.
| Charm Type | Charm Name | Channel | Status/Recommendation |
| ----- | ----- | ----- | ----- |
| **Machine Charm - Reactive** | [canonical-livepatch-server](https://charmhub.io/canonical-livepatch-server) | `latest/*` | Reactive charm (deprecated) |
| **Machine Charm - Operator** | [canonical-livepatch-server](https://charmhub.io/canonical-livepatch-server) | `ops1.x/*` | Operator charm (deprecated) |
| **Kubernetes Charm - Operator** | [canonical-livepatch-server-k8s](https://charmhub.io/canonical-livepatch-server-k8s) | `latest/*` | Operator charm (recommended for new deployments) |

This guide specifically describes how to migrate from the Reactive charm (in channel `latest/*`) to the Livepatch server snap package. For simplicity, this guide uses the same host instance as the juju deployment, to deploy the Livepatch server snap.

## Migrate Configuration

The new Livepatch server operator charms and snaps have different configuration keys when compared to the reactive charms. The configuration was restructured to be simple and easy to read. As a result, migrating the configuration from the reactive charm to the snap is not straightforward. However, we provide a tool with the Livepatch server snap that simplifies the process. [Refer to the "Migrate Configuration" table](/livepatch_on_prem/how-to-guides/migrate-from-reactive-charm-to-operator-charm.md) to understand how the configuration keys have changed.

1. To access the migration tool, install the [canonical-livepatch-server snap](https://snapcraft.io/canonical-livepatch-server). This is the same snap that will be used later to set up the Livepatch server.

   ```shell
   sudo snap install canonical-livepatch-server --channel=latest/stable
   ```

   Ensure that the snap version is at least `v1.17.18` in order to be able to use the migration tool.

   ```shell
   snap list | grep canonical-livepatch-server
   ```

2. Save the reactive machine charm configuration of the Livepatch server deployment in a yaml file.

   ```shell
   juju config <livepatch-application-name> > old-config.yaml
   ```

3. Move the configuration file to the $SNAP_COMMON directory of the Livepatch server snap. This needs to be done because the Livepatch server snap is strictly confined and cannot access files outside of its snap specific directories.

   ```shell
   sudo mv old-config.yaml /var/snap/canonical-livepatch-server/common/
   ```

4. Use the migrate-config tool available with the Livepatch server snap. You can use this command to display the migrated configuration, dry-run the application of the configuration and to apply the migrated configuration to the Livepatch server snap deployment. All the configuration values that were set in the reactive charm will be migrated to the snap. Configuration options with empty values will not be migrated to prevent overwriting the default configuration values present in the snap configuration.

   - Display the migrated configuration and save the new configuration to a file. This step requires sudo/root user access to be able to create a new output file in the $SNAP_COMMON directory of the Livepatch server snap. This can be used to verify the configuration before applying. (Optional)

     ```shell
     # Use this to display the new configuration in the terminal
     sudo canonical-livepatch-server.migrate-config \
     -i /var/snap/canonical-livepatch-server/common/old-config.yaml

     # Use this to save the new configuration to a file in $SNAP_COMMON
     sudo canonical-livepatch-server.migrate-config \
     -i /var/snap/canonical-livepatch-server/common/old-config.yaml -o
     ```

   - Dry run the migration of the configuration. (Optional)

     ```shell
     canonical-livepatch-server.migrate-config \
     -i /var/snap/canonical-livepatch-server/common/old-config.yaml --set-config --dry-run
     ```

   - Set the snap configuration to the migrated configuration values. This step requires sudo/root user access to be able to set the snap configuration.

     ```shell
     sudo canonical-livepatch-server.migrate-config \
     -i /var/snap/canonical-livepatch-server/common/old-config.yaml --set-config
     ```

   5. Review the new configuration of the Livepatch server snap and make modifications where needed. Refer to the [configuration documentation](/livepatch_on_prem/reference/configuration.md) for more information.

   - To view the new configuration run:

     ```shell
     snap get canonical-livepatch-server -d lp
     ```

   - To modify a configuration option, if necessary, run:

     ```shell
     sudo snap set canonical-livepatch-server lp.<key>=<value>
     ```

(livepatch_on_prem-how-to-guides-database-migration)=

## Database Migration

Data migration from the [PostgreSQL database machine charm](https://charmhub.io/postgresql) used by the Livepatch server reactive charm, to the database used by the Livepatch server snap is essential to preserve the machine and patch data that has already been stored. There are however, a few nuances in migrating this data from the charm to the snap. We do not want to preserve the roles and ownership created by the PostgreSQL charm, because these roles only make sense in the context of charms and Juju. This is taken into account when describing the steps for data migration below.

````{note}
It is assumed that the PostgreSQL database was deployed using the [machine charm](https://charmhub.io/postgresql) to interact with the Livepatch server charm. For the Livepatch server snap, deploy PostgreSQL in a suitable production environment. **For simplicity, in this guide, the Livepatch server snap connects to a PostgreSQL instance running in a Docker container. This is not recommended for using PostgreSQL in production environments.**

```shell
docker run \
--name postgresql \
-e POSTGRES_USER=livepatch \
-e POSTGRES_PASSWORD=testing \
-p 5432:5432 \
-d postgres:14
```

This setup can differ on a case by case basis and would result in slightly different steps while migrating the data.

````

1. Download the tools necessary for PostgreSQL database migration.
   ```shell
   sudo apt install postgresql-client postgresql-client-common
   ```

2. Obtain the IP address of the primary PostgreSQL database unit using the output of `juju status`.
   |Unit|Workload|Machine|Public address|Ports| Message|
   |------|----------|--------------|----------------------|--------|------------|
   |postgresql/0\*| active| 1 | 10.239.140.105| 5432/tcp | Primary

3. Get the system user’s password for the PostgreSQL database charm unit. The password is obtained by using the [\`get-password\` action](https://charmhub.io/postgresql/actions#get-password) defined by the PostgreSQL machine charm. The action gets the password for the `operator` username by default. The action must be run for the unit configured to be the primary database.

   ```shell
   juju run postgresql/0 get-password
   ```

4. Dump the database data from the PostgreSQL unit, using the [pg_dump tool](https://www.postgresql.org/docs/14/app-pgdump.html). The command below will prompt the `operator` user for a password, at which point the password obtained from the previous step needs to be entered.

   ```shell
   pg_dump -Fc livepatch -h <postgres-IP> -U operator > dump-file
   ```

```{note}
If the reactive charm deployment of the Livepatch server uses the \`filesystem\` patch storage  type, the database dump step might be a little different. Refer the [Patch Migration section](#livepatch_on_prem-how-to-guides-patch-migration) below for more information on a different command to run for the database dump.
```

5. Copy the dump file to an environment accessible by the new PostgreSQL database deployment. In our case, the dump file will be copied to the docker container running the database i.e. container with name `postgresql`.

   ```shell
   docker cp dump-file postgresql:/dump_file
   ```

6. Restore the data from the dump file to the new database, using the [pg_restore tool](https://www.postgresql.org/docs/14/app-pgrestore.html). Here, we use the no-owner (-O) and no-privileges (-x) options to prevent restoration of owners and privileges from the old postgres database. This is done to avoid migrating over owners that only make sense in the context of charms and Juju. The pg_restore is done within the docker container.

   ```shell
   docker exec -it postgresql bash

   pg_restore dump-file -d livepatch -U livepatch -Ox
   ```

7. The final step involves running schema upgrades on the database, as the database used by the new Livepatch server versions have a different schema than the one used by the Livepatch server reactive charm. This is a very important step, without which the Livepatch server snap will fail. This command uses the schema-tool provided by the Livepatch server snap, which accepts the database connection string as the argument and applies the schema upgrades.

   ```shell
   canonical-livepatch-server.schema-tool \
   postgresql://livepatch:testing@localhost:5432/livepatch
   ```

After successfully completing these steps, the new PostgreSQL database will contain all the data present in the PostgreSQL charm. The next section explains how the patches synced from the upstream Livepatch server can be migrated to be used by the Livepatch server snap.

(livepatch_on_prem-how-to-guides-patch-migration)=

## Patch Migration

Data and Patch Migration are closely related because the type of patch storage used by the Livepatch server, running as a reactive machine charm, would define which migration steps are necessary (or not), so that these patches and their corresponding data are effectively migrated to the Livepatch server snap. Note that, only the patch data migration would need additional steps depending on the type of patch storage used. All other data stored by the Livepatch server in the PostgreSQL database can be directly migrated to the new database used by the Livepatch server snap.

Let us consider the different patch storage types and the migration steps necessary to migrate the patches and data.

1. **PostgreSQL**
   The `postgres` patch storage type implies that the patches were stored in a postgres database in the `patch_file_data` table. In this case, patch migration does not need any extra steps. Migrating the data from the PostgreSQL database used by the reactive charm to the PostgreSQL database used by the snap is sufficient, for patch migration. If a dedicated PostgreSQL database is being used for patch storage, follow the exact steps for database migration as mentioned above, to also migrate the database containing the patches.\
   The `patch-storage.postgres-connection-string` configuration of the Livepatch server snap, needs to be set with the connection string of the PostgreSQL database containing the migrated patches.

   ```shell
   sudo snap set canonical-livepatch-server \
   lp.patch-storage.postgres-connection-string=<postgres-database-connection-string>
   ```

2. **Swift and S3**\
   The `swift` and `s3` patch storage types imply that the patches are stored in a remote AWS S3 or Swift bucket. This means that the migration of the configuration values and database migration from the reactive charm to the server snap is sufficient. Only the network connectivity between the machine running the Livepatch server snap and the remote bucket should be verified. This might require modification of firewall rules depending on the setup.

3. **Filesystem**\
   The `filesystem` patch storage type implies that the patch files were stored in a filesystem accessible by the Livepatch server reactive charm. Migrating patches stored in a filesystem could be slightly more complex depending on the permissions and accessibility of the filesystem.

- If the filesystem where the patches are stored is accessible, and the patches can be easily moved from the machine on which the Livepatch server is running as a reactive charm, to the machine running the Livepatch server snap, the patch migration process is straightforward. This method saves us the time, resources and effort of redownloading all the patches from the upstream Livepatch server.

  - For this example, we will consider that the Livepatch server reactive charm is running on an LXD container and the Livepatch server snap runs on the host machine running the LXD container. The patches are stored in the /livepatch directory of the LXD container. The patch migration process involves copying the patches to the Livepatch server snap machine, and then moving the patches to the snap’s [$SNAP_COMMON](https://snapcraft.io/docs/environment-variables#snap-common) directory.
    ```shell
    # Pull patch files from the LXD container
    sudo lxc file pull <lxc-container-name> /livepatch/ \
    /var/snap/canonical-livepatch-server/common/ -pr

    # Move the files from /livepatch to /patches (default file system path for the snap)
    sudo mv /var/snap/canonical-livepatch-server/common/livepatch/* \
    /var/snap/canonical-livepatch-server/common/patches
    ```

- If the filesystem where the patches are stored is inaccessible and cannot be moved to the new machine, the patch migration process gets a little more complicated. The process involves preventing the migration of the patch data during the database migration, and instead syncing the patches from the upstream Livepatch server.

  - To prevent the migration of the patch data, run the following command in place of the pg_dump command shown in the database migration section.

    ```shell
    pg_dump -Fc livepatch -h <postgres-IP> -U operator \
    --exclude-table-data="patch" --exclude-table-data="patch_file" \
    --exclude-table-data="patch_file_tier" --exclude-table-data="kernel" \
    > dump-file
    ```

  - This command will ensure that the patch file data in specific tables is not migrated. The rest of the steps for database migration are the same as described in the [Database migration section](#livepatch_on_prem-how-to-guides-database-migration).

  - This enables users to sync patches from the upstream Livepatch server and fill the patch file data in the database. The steps for patch synchronization and populating patch file data are described in the next section.

## Using the livepatch-admin tool for patch file and storage synchronization

The livepatch-admin tool is useful for syncing patches from the upstream Livepatch server and populating the database with the synced patch data. Follow the how-to guides mentioned below to set up the livepatch-admin tool and synchronize patches from the hosted Livepatch server.

1. [How to set up the livepatch-admin tool](/livepatch_on_prem/how-to-guides/setup-administration-tool.md)
2. [How to fetch patches from the hosted Livepatch server](/livepatch_on_prem/how-to-guides/fetch-patches.md)

Once the patches have been downloaded, run `livepatch-admin storage refresh` to sync the patch storage and patch data in the database.

```{note}
It is recommended to run `livepatch-admin storage refresh` after the database and patch migration, irrespective of the type of patch storage and how the migration was done, as it helps confirm that both the patch data in the database and the patches in the storage are in sync.
```
