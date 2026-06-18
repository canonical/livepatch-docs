---
myst:
  html_meta:
    description: "How to deploy the Livepatch server snap on public clouds."
---


(server-how-to-guides-deployment-public-clouds)=

# How to deploy on public clouds

This section details how to launch the Livepatch server snap on public clouds, with single-unit or multi-unit deployments using an auto-scaling solution from the cloud provider.

## Prerequisites

Before deploying the server snap, the following are required:

* Two or more VMs with at least Ubuntu 22.04 LTS, with at least two CPU cores and at least 4 GB of RAM. Between two and four VMs are sufficient.
* A PostgreSQL DNS for an instance with a database ready to be initialized.
* A secure vault for storing secrets: an AWS IAM role or Azure Key Vault.
* A storage solution: S3, Ceph, or a PostgreSQL connection string.
* An Ubuntu Pro token from [ubuntu.com/pro/dashboard](https://ubuntu.com/pro/dashboard).
* Optional: the [CVE service](https://snapcraft.io/canonical-livepatch-cve-service) deployed and ready for connections.

> This setup is incompatible with Ubuntu Core.

Sensitive data such as connection strings and the Pro token should be stored in a vault that is accessed later during the installation.

## Install Livepatch server with cloud-init

VMs can be created and Livepatch can be initialized with [cloud-init](https://docs.cloud-init.io/en/latest/). Create a `cloud-config.yaml` file with the following template, including setup steps to install the Livepatch server:

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - snapd
  - jq

write_files:
  - path: /etc/livepatch/livepatch.env
    permissions: "0600"
    owner: root:root
    content: |
      export URLTEMPLATE="http://<address>/v1/patches/{filename}"
  - path: /etc/livepatch/database_url
    permissions: "0600"
    owner: root:root
  - path: /etc/livepatch/pro_token
    permissions: "0600"
    owner: root:root
  - path: /etc/livepatch/livepatch_admin_user
    permissions: "0600"
    owner: root:root
    # Storage option
       ...

runcmd:
  - |
    set -e
    snap wait system seed.loaded
    snap install aws-cli --classic
    . /etc/livepatch/livepatch.env        
    # setup PostgreSQL connection  
    ...    
    # set pro token
    ...    
    # setup storage
    ...  
    # setup user
    ...
  - |
    set -e
    snap install canonical-livepatch-server
    snap refresh --hold canonical-livepatch-server
    canonical-livepatch-server.schema-tool "$(cat /etc/livepatch/database_url)"      
    
    snap set canonical-livepatch-server \
    token="$(cat /etc/livepatch/pro_token)" \
    lp.database.connection-string="$(cat /etc/livepatch/database_url)" \
    lp.server.url-template=$URLTEMPLATE \
    lp.auth.basic.enabled=true \
    lp.patch-sync.enabled=true \
    lp.patch-sync.interval=12h \
    lp.auth.basic.users="$(cat /etc/livepatch/livepatch_admin_user)" \
    lp.server.server-address=0.0.0.0:80 \ 
    # optional, if you have the CVE snap deployed
    lp.cve-lookup.enabled="true" \
    lp.cve-sync.enabled="true" \
    lp.cve-sync.source-url="http(s)://<host>:port"
  - |
    rm /etc/livepatch/*


final_message: The system is up, up to date, and Livepatch server is active after $UPTIME second
```

This template configuration performs the following operations:

* Waits until snapd is ready.
* Installs the `canonical-livepatch-server` snap and stops the server temporarily for configuration.
* Initializes the provided database with the latest schema. If the database schema is already up to date, no updates are performed.
* Points the Livepatch server to the configured database.
* Enables the server with an Ubuntu Pro token.
* Sets up an admin user.
* Configures the server for all traffic on port 80.
* Starts the server with the provided configuration.
* Cleans up temporary files used in configuration.

This cloud-init module expects to find required parameters (secrets, user strings, database DNS) in root-only files located at `/etc/livepatch/` during boot time. The method for obtaining the required secrets is unique to the cloud environment. The recommended approach is to use the cloud's vault provider to securely access the secrets and write them to the required locations. The files are deleted during the final step of the cloud-init setup.

## Set up authentication

In the configuration, basic authentication can be set up and a user can be added for the admin tool with these commands:

```bash
snap set canonical-livepatch-server lp.auth.basic.enabled=true
snap set canonical-livepatch-server lp.auth.basic.users="$(cat /etc/livepatch/livepatch_admin_user)"
```

Where the user and hashed password in the form of `username:hashedpassword` are fetched from a secure vault and written to a root-only file at `/etc/livepatch/livepatch_admin_user`.

To ensure the file is root-accessible only, add a field in the `write_files` section:

```yaml
  - path: /etc/livepatch/livepatch_admin_user
    permissions: "0600"
    owner: root:root
```

> The Livepatch server requires the Pro token, database connection string, and storage connection string to operate. Do not put these values in the configuration file as plain text. Use the cloud provider's secrets manager.

## Set up periodic patch sync

Periodic patch syncs with the hosted Livepatch server can be configured with:

```bash
snap set canonical-livepatch-server lp.patch-sync.enabled=true
snap set canonical-livepatch-server lp.patch-sync.interval=12h
```

All configuration can be performed with a single `snap set` command by placing all configuration values in a single line, as shown in the template cloud-init module.

With all configurations written in the `cloud-config.yaml`, VM instances can be created by passing in the configuration. The Livepatch server is installed and ready to receive traffic.

```{toctree}
:titlesonly:
:maxdepth: 2
:glob:
:hidden:

Deploying on aws <deploying-on-aws.md>
Deploying on azure <deploying-on-azure.md>
```
