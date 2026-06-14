---
myst:
  html_meta:
    description: "How to deploying on azure on public clouds with Livepatch server."
---


(livepatch_server_on_public_clouds-how-to-guides-deploying-the-livepatch-server-snap-on-azure)=

# Deploying the Livepatch Server snap on Azure

This section details how to deploy the Livepatch Server snap in auto-scaling configuration on Azure.

## Required Resources

To set up Livepatch server on Azure using an Filesystem or PostgreSQL (Azure Blob Store is not yet supported) for patch storage, you will need:

- A VNet with subnets for VMSS, Application Gateway and PostgreSQL DB.
- An Azure Key Vault with access enabled for the VMSS subnet of the VNet.
- An Azure-managed PostgreSQL instance with PostgreSQL 14 or above using password authentication.
  - Username and password stored in Azure Key Vault as a JSON object.
- A Virtual Machine ScaleSet (VMSS) with an instance type with at least 2 vCPUs, 4 GB RAM with cloud-init config file as custom data.
- An Ubuntu Pro token stored as a secret in Azure Key Vault.
- Livepatch server admin credentials stored in Azure Key Vault Secrets in `<username>:<hashedpassword>` format.
- A Managed Identity with the `Key Vault Secrets User` role in Azure Key Vault.
- An Application Gateway with a public IP address as an entry point for accessing Livepatch server.
- A VMSS as backend pool with the corresponding Managed Identity attached to it.

## Creating The Deployment

In Azure, Livepatch can be deployed on VMSS behind an Application Gateway with autoscaling configured on VMSS instances. The VMSS instances can be configured to auto-install Livepatch server snap at startup using cloud-init configuration file in a Standard Ubuntu Server image. This guide uses PostgreSQL server as patch storage.

Configure a VNet with 3 different subnets, one for each: Application Gateway, VMSS, DB. The Application Gateway subnet should have at least 256 addresses.

Create an Azure-managed PostgreSQL DB with PostgreSQL password authentication. Disable public access and use the DB subnet from the VNet to provision the DB. Create a database in this PostgreSQL server for Livepatch Server to use (this can be done from Azure UI).

Create a Managed Identity. This will be assigned to the VMSS to allow access to Azure Key Vault.

Create an Azure Key Vault with Public Access disabled. Allow access from the VMSS subnet of the VNet. Store the DB credentials (JSON format), admin credentials (`<username>:<hashedpassword>` format) and pro token (raw string) in the Key vault. Assign “Key Vault Secrets User” role for this Vault to the Managed Identity.

For VMSS, Select an instance type that has sufficient CPU and memory capacity. The minimum resources required for Livepatch server to run efficiently are 2 vCPU and 4 GB memory. Avoid using B-series instances. For the storage options, the default volume size of 30GB will be sufficient. Assign the Managed Identity to the VMSS to allow access to Azure Key Vault. Configure the VMSS NSG to allow traffic on port 80 from within the VNet. It is recommended to not have public IP addresses assigned to the VMSS VMs. This will ensure that VMs can be accessed only via Application Gateway. Provision a jumpbox in the same VNet with public IP to access VMs, if and when required. Add the following cloud-init config file to the CustomData field:

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - snapd
  - jq

write_files:
  - path: /etc/client/livepatch.env
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
  - path: /etc/client/database_url
    permissions: "0600"
    owner: root:root
  - path: /etc/client/pro_token
    permissions: "0600"
    owner: root:root
  - path: /etc/client/livepatch_admin_user
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

    . /etc/client/livepatch.env
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
     > /etc/client/database_url

    # set pro token
    az keyvault secret show --id $TOKEN_SECRET_ID | jq -r '.value' > /etc/client/pro_token

    # setup user
    az keyvault secret show --id $ADMIN_USER_SECRET_ID | jq -r '.value' > /etc/client/livepatch_admin_user

  - |
    set -e
    snap wait system seed.loaded
    snap stop canonical-livepatch-server.livepatch
    canonical-livepatch-server.schema-tool "$(cat /etc/client/database_url)"


    snap set canonical-livepatch-server \
    token="$(cat /etc/client/pro_token)" \
    lp.database.connection-string="$(cat /etc/client/database_url)" \
    lp.patch-storage.type=postgres \
    lp.server.url-template=$URLTEMPLATE \
    lp.patch-storage.postgres-connection-string="$(cat /etc/client/database_url)" \
    lp.auth.basic.enabled=true \
    lp.patch-sync.enabled=true \
    lp.patch-sync.interval=12h \
    lp.auth.basic.users="$(cat /etc/client/livepatch_admin_user)" \
    lp.server.server-address=0.0.0.0:80

    snap start canonical-livepatch-server.livepatch
  - |
    rm /etc/client/*

final_message: The system is up, up to date, and Livepatch server is active after $UPTIME second
```

Ensure that correct ENV var values are filled in the `write-files` section of the config.

Provision a Standard V2 (or WAF V2) Application Gateway in the subnet created in the VNet with autoscaling enabled. The minimum instance count for the same should be set according to the expected traffic. Add the VMSS to the backend pool of the Application Gateway and route all traffic to this pool. Use port 80 with `GET /` as the health check endpoint for the VMSS.

Autoscaling on VMSS should be configured based on both CPU and RAM metrics.

## Troubleshooting

You can ssh into VMSS instances to check livepatch server status. You can check if Livepatch server snap logs by running:

```shell
sudo snap logs canonical-livepatch-server
```

If there are no error logs, then the server has successfully initialized. If there are errors, check the status of

```shell
cloud-init status --long
```

If the status shows: **error**, then something went wrong during the cloud-init procedure. You can view the command output logs at `/var/log/cloud-init-output.log`.
