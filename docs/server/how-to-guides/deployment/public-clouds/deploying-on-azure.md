---
myst:
  html_meta:
    description: "How to deploy the Livepatch server snap on Azure."
---


(server-how-to-guides-deploying-the-livepatch-server-snap-on-azure)=

# How to deploy on Azure

This section details how to deploy the Livepatch Server snap in auto-scaling configuration on Azure.

## Required resources

To set up the Livepatch server on Azure using a filesystem or PostgreSQL (Azure Blob Storage is not yet supported) for patch storage, the following are required:

* A VNet with subnets for VMSS, Application Gateway, and PostgreSQL DB.
* An Azure Key Vault with access enabled for the VMSS subnet of the VNet.
* An Azure-managed PostgreSQL instance with PostgreSQL 14 or above using password authentication. The username and password stored in Azure Key Vault as a JSON object.
* A Virtual Machine Scale Set (VMSS) with an instance type with at least two vCPUs and 4 GB RAM, with a cloud-init configuration file as custom data.
* An Ubuntu Pro token stored as a secret in Azure Key Vault.
* Livepatch server admin credentials stored in Azure Key Vault Secrets in `<username>:<hashedpassword>` format.
* A Managed Identity with the `Key Vault Secrets User` role in Azure Key Vault.
* An Application Gateway with a public IP address as an entry point for accessing the Livepatch server.
* A VMSS as a backend pool with the corresponding Managed Identity attached.

## Create the deployment

In Azure, Livepatch can be deployed on VMSS behind an Application Gateway with autoscaling configured on VMSS instances. The VMSS instances can be configured to auto-install the Livepatch server snap at startup using a cloud-init configuration file in a Standard Ubuntu Server image. This guide uses a PostgreSQL server for patch storage.

### Configure the VNet

Configure a VNet with three different subnets, one for each: Application Gateway, VMSS, and DB. The Application Gateway subnet should have at least 256 addresses.

### Create the PostgreSQL database

Create an Azure-managed PostgreSQL DB with PostgreSQL password authentication. Disable public access and use the DB subnet from the VNet to provision the database. Create a database in this PostgreSQL server for the Livepatch server to use. This can be performed from the Azure UI.

### Create the Managed Identity

Create a Managed Identity. This will be assigned to the VMSS to allow access to Azure Key Vault.

### Create the Azure Key Vault

Create an Azure Key Vault with public access disabled. Allow access from the VMSS subnet of the VNet. Store the DB credentials (JSON format), admin credentials (`<username>:<hashedpassword>` format), and Pro token (raw string) in the Key Vault. Assign the `Key Vault Secrets User` role for this Vault to the Managed Identity.

### Configure the VMSS

For VMSS, select an instance type with sufficient CPU and memory capacity. The minimum resources required for the Livepatch server to run efficiently are two vCPUs and 4 GB memory. Avoid using B-series instances. For storage options, the default volume size of 30 GB is sufficient. Assign the Managed Identity to the VMSS to allow access to Azure Key Vault.

Configure the VMSS NSG to allow traffic on port 80 from within the VNet. It is recommended not to have public IP addresses assigned to the VMSS VMs. This ensures that VMs can be accessed only through the Application Gateway. Provision a jumpbox in the same VNet with a public IP to access VMs when required.

Add the following cloud-init configuration file to the Custom Data field:

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - snapd
  - jq

write_files:
  - path: /etc/livepatch/livepatch.env
    permissions: '0600'
    owner: root:root
    content: |
      # azure variables
      export AZURE_SUBSCRIPTION_ID=<azure subscription id>
      export AZURE_RESOURCE_GROUP=<azure resource group>
      export AZURE_LOCATION=<azure location>

      # database variables
      export DB_ENDPOINT=<db endpoint>
      export DB_PORT=5432
      export DB_SECRET_ID=<db secret id>
      export DB_NAME=<db name>

      # pro token variables
      export TOKEN_SECRET_ID=<token secret id>
      export ADMIN_USER_SECRET_ID=<admin user secret id>

      # storage variables
      export URLTEMPLATE=https://<your-domain>/v1/patches/{filename}

      # managed identity variable
      export AZURE_MANAGED_IDENTITY_CLIENT_ID=<azure managed identity client-id>
  - path: /etc/livepatch/database_url
    permissions: "0600"
    owner: root:root
  - path: /etc/livepatch/pro_token
    permissions: "0600"
    owner: root:root
  - path: /etc/livepatch/livepatch_admin_user
    permissions: "0600"
    owner: root:root

snap:
  commands:
    - snap install canonical-livepatch-server
    - snap refresh --hold canonical-livepatch-server

runcmd:
  - |
    set -e

    # Install azure-cli (https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux?view=azure-cli-latest&pivots=apt)
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
    sudo mkdir -p /etc/apt/keyrings
    curl -sLS https://packages.microsoft.com/keys/microsoft.asc |
      gpg --dearmor | sudo tee /etc/apt/keyrings/microsoft.gpg > /dev/null
    sudo chmod go+r /etc/apt/keyrings/microsoft.gpg
    AZ_DIST=$(lsb_release -cs)
    echo "Types: deb
    URIs: https://packages.microsoft.com/repos/azure-cli/
    Suites: ${AZ_DIST}
    Components: main
    Architectures: $(dpkg --print-architecture)
    Signed-by: /etc/apt/keyrings/microsoft.gpg" | sudo tee /etc/apt/sources.list.d/azure-cli.sources
    sudo apt-get update
    sudo apt-get install -y azure-cli

    . /etc/livepatch/livepatch.env
    # setup azure cli
    az login -i --client-id $AZURE_MANAGED_IDENTITY_CLIENT_ID
    az account set -s $AZURE_SUBSCRIPTION_ID
    export AZURE_DEFAULTS_GROUP=$AZURE_RESOURCE_GROUP
    export AZURE_DEFAULTS_LOCATION=$AZURE_LOCATION

    # setup PostgreSQL connection
    az keyvault secret show --id $DB_SECRET_ID | \
    jq -r \
     --arg DB_ENDPOINT "$DB_ENDPOINT" \
     --arg DB_PORT "$DB_PORT" \
     --arg DB_NAME "$DB_NAME" \
     '.value | fromjson | "postgresql://\(.username):\(.password | @uri)@\($DB_ENDPOINT):\($DB_PORT)/\($DB_NAME)"' \
     > /etc/livepatch/database_url

    # set pro token
    az keyvault secret show --id $TOKEN_SECRET_ID | jq -r '.value' > /etc/livepatch/pro_token

    # setup user
    az keyvault secret show --id $ADMIN_USER_SECRET_ID | jq -r '.value' > /etc/livepatch/livepatch_admin_user

  - |
    set -e
    snap wait system seed.loaded
    snap stop canonical-livepatch-server.livepatch
    canonical-livepatch-server.schema-tool "$(cat /etc/livepatch/database_url)"


    snap set canonical-livepatch-server \
    token="$(cat /etc/livepatch/pro_token)" \
    lp.database.connection-string="$(cat /etc/livepatch/database_url)" \
    lp.patch-storage.type=postgres \
    lp.server.url-template=$URLTEMPLATE \
    lp.patch-storage.postgres-connection-string="$(cat /etc/livepatch/database_url)" \
    lp.auth.basic.enabled=true \
    lp.patch-sync.enabled=true \
    lp.patch-sync.interval=12h \
    lp.auth.basic.users="$(cat /etc/livepatch/livepatch_admin_user)" \
    lp.server.server-address=0.0.0.0:80

    snap start canonical-livepatch-server.livepatch
  - |
    rm /etc/livepatch/*

final_message: The system is up, up to date, and Livepatch server is active after $UPTIME second
```

Ensure the correct environment variable values are filled in the `write_files` section of the configuration.

### Provision the Application Gateway

Provision a Standard V2 (or WAF V2) Application Gateway in the subnet created in the VNet with autoscaling enabled. The minimum instance count should be set according to the expected traffic. Add the VMSS to the backend pool of the Application Gateway and route all traffic to this pool. Use port 80 with `GET /` as the health check endpoint for the VMSS.

Autoscaling on VMSS should be configured based on both CPU and RAM metrics.

## Troubleshooting

SSH into VMSS instances to check the Livepatch server status. Check the Livepatch server snap logs by running:

```shell
sudo snap logs canonical-livepatch-server
```

If there are no error logs, the server has initialized successfully. If there are errors, check the status of `cloud-init`:

```shell
cloud-init status --long
```

If the status shows **error**, something went wrong during the cloud-init procedure. The command output logs can be viewed at `/var/log/cloud-init-output.log`.
