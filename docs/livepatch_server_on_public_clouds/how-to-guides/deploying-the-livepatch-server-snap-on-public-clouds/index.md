---
myst:
  html_meta:
    description: "How to deploying-the-livepatch-server-snap-on-public-clouds on public clouds with Livepatch server."
---


(livepatch_server_on_public_clouds-how-to-guides-deploying-the-livepatch-server-snap-on-public-clouds)=

# Deploying The Livepatch Server Snap On Public Clouds

This section details how to launch the Livepatch server snap on public clouds, with single or multi unit deployments using an auto scaling solution from the cloud provider.

## Prerequisites

Before deploying the server snap, you will need the following:

- Two or more VMs with at least Ubuntu 22.04 LTS, with at least 2 CPU cores and at least 4GB of RAM (between two and four VMs will suffice).
- A PostgreSQL DNS for an instance with a database ready to be initialized.
- A secure vault for storing secrets (via an AWS IAM role, or Azure Key Vault)
- A storage solution: S3, Ceph, or PostgreSQL connection string.
- An Ubuntu Pro token from [ubuntu.com/pro/dashboard](https://ubuntu.com/pro/dashboard).
- Optional: the [CVE service](https://snapcraft.io/canonical-livepatch-cve-service) deployed and ready for connections.

> Note: This setup is incompatible with Ubuntu core.

Sensitive data such as connection strings, and the pro token, should be stored in a vault we will access later during the installation.

## Livepatch server installation with cloud-init

You can easily create VMs and initialize Livepatch with [cloud-init](https://docs.cloud-init.io/en/latest/): create a `cloud-config.yaml` file with the following template including setup steps to install Livepatch server:

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

This template config performs the following commands:

- Waits until snapd is ready.
- Installs the canonical-livepatch-server snap and stops the server temporarily for configuration.
- Initializes the provided database with the latest schema (if the database schema is up to date, this operation won’t perform updates).
- Points Livepatch server to the configured database
- Enables the server with an Ubuntu Pro token.
- Sets up an admin user.
- Sets the server for all traffic on port 80.
- Starts the server with the provided configuration.
- Cleans up temporary files used in configuration.

This cloud-init module expects to find required parameters (secrets, user strings, database DNS) in root-only files located at `/etc/livepatch/` during boot time. Specifics on getting required secrets are unique to the cloud environment. We recommend using the cloud’s vault provider to securely access the secrets and write them to the required locations. The files are deleted during the last step of the cloud-init setup.

In the config, you may also want to setup basic authentication, and add a user for the admin tool with these commands:

```bash
snap set canonical-livepatch-server lp.auth.basic.enabled=true
snap set canonical-livepatch-server lp.auth.basic.users="$(cat /etc/livepatch/livepatch_admin_user)"
```

Where the user and hashed password in the form of _**username:hashedpassword**_ are fetched from a secure vault and written to a root-only file `/etc/livepatch/livepatch_admin_user`.

Make sure to make this file root accessible only by adding a field in the write_files section:

```yaml
  - path: /etc/livepatch/livepatch_admin_user
    permissions: "0600"
    owner: root:root
```

> Note: Livepatch server requires the pro token, database connection string, and storage connection string to operate, but do not put these values in the configuration file as plain text, use your cloud provider's secrets manager.

Additionally, you can set up periodic patch syncs with the hosted Livepatch server with:

```bash
snap set canonical-livepatch-server lp.patch-sync.enabled=true
snap set canonical-livepatch-server lp.patch-sync.interval=12h
```

All configuration can be done with a single snap set command by putting all configuration values in a single line, as done in the template cloud-init module.

With all the configurations written in the cloud-config.yaml you can easily create VM instances by passing in the config with Livepatch server installed and ready to receive traffic.

```{toctree}
:titlesonly:
:maxdepth: 2
:glob:
:hidden:

deploying-on-aws
deploying-on-azure
```
