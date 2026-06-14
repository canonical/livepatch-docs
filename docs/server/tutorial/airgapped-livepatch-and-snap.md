---
myst:
  html_meta:
    description: "Tutorial: Air-gapped Livepatch and Snap - hands-on introduction to Livepatch on-prem."
---

(server-tutorial-airgapped-livepatch-and-snap)=

# Airgapped Livepatch and Snap

## Introduction

Livepatch on-prem is a self-hosted version of the Livepatch server, enabling the delivery of patches to machines within network restricted environments. For security reasons, administrators may prefer to deploy Livepatch on-prem server in an airgapped environment with restricted Internet access.

This tutorial will deploy the Livepatch on-prem server as a Snap package in an airgapped environment.

### How does Livepatch on-prem work in an airgapped environment?

Generally, in order to perform authentication/authorisation of machines and to fetch patches, the Livepatch on-prem server needs to communicate with the main Livepatch server hosted by Canonical. In an airgapped environment, where such communication is not available, these functions are handled using the following tools:

- [**Airgapped Ubuntu Pro Server**](https://discourse.charmhub.io/t/15278) provides services related to Ubuntu Pro subscriptions in airgapped environments. Livepatch on-prem can be integrated with this service to perform authentication/authorisation of machines and handle subscription-related functionality.
- [**Patch Downloader**](https://snapcraft.io/canonical-livepatch-downloader) is a CLI tool that can be used to download the latest patch files from the Livepatch server. In an airgapped setup, the administrators of Livepatch on-prem should use this tool to fetch the latest patches and then upload them to the configured patch storage. You can check out [this](/server/reference/patch-storage/index) topic on how to configure various types of storage for Livepatch on-prem. [This](/server/how-to-guides/patch-management/use-the-patch-downloader-tool) topic explains how to use the Patch Downloader tool to fetch patches.

```{note}
When deploying airgapped Livepatch on-prem using Snap, it is best to configure the patch storage to something independently accessible within your infrastructure, like the filesystem or an S3/Swift bucket, instead of PostgreSQL. This way, Livepatch administrators can independently download the latest patches via the Patch Downloader CLI tool and transfer them to the patch storage.
```

## Deployment steps

In this tutorial, we use [Multipass](https://multipass.run) to create Ubuntu virtual machines (VM) to deploy Livepatch and its dependencies on it.

If not already installed, we can use the command below to install Multipass:

```sh
sudo snap install multipass
```

Also, we will need two Multipass virtual machines; one as the airgapped environment where the Livepatch on-prem server is going to be deployed, and the other to simulate a normal environment (i.e., with access to the Internet) to finalize the configurations for the airgapped Ubuntu Pro server.

### Step 1: Create Multipass instances

You can create the Multipass instances needed in this tutorial by running the command below. This will create two instances, named `pro-configuration` and `livepatch-deploy`.

```sh
multipass launch jammy --name pro-configuration
multipass launch jammy --name livepatch-deploy -d 10G
```

### Step 2: Create configurations for the airgapped Ubuntu Pro server

We first need an interactive shell in the `pro-configuration` instance:

```sh
multipass shell pro-configuration
```

Once getting into the instance, we need to install the `pro-airgapped` configuration tool:

```sh
sudo add-apt-repository ppa:yellow/ua-airgapped
sudo apt update
sudo apt install pro-airgapped
```

To create the configuration file, you will need your Ubuntu Pro subscription token. You should copy your token from the Ubuntu Pro [dashboard](https://ubuntu.com/pro/dashboard), replace the `<TOKEN>` placeholder in the following command, and then run the command:

```sh
cat <<EOF > override.yml
<TOKEN>:
  livepatch:
    directives:
      remoteServer: http://livepatch.test.com:8080
  livepatch-onprem:
    directives:
      remoteServer: http://livepatch.test.com:8080
EOF
```

```{note}
Here we have set the Livepatch on-prem server hostname to `livepatch.test.com`. You can set it to any other value, but remember to replace it in the next steps.
```

This will create a file named `override.yml`. Now, we should use the `pro-airgapped` tool to make the final configuration file, which we will use to set up the airgapped environment. Note that the `pro-airgapped` tool needs Internet access to communicate with upstream Canonical services to fetch your subscription details. By running the following command the final configuration file will be created as `server-ready.yml`:

```sh
cat override.yml | pro-airgapped > server-ready.yml
```

Now, we are done with this Multipass instance, and we should exit the interactive shell:

```sh
exit
```

### Step 3: Transfer configuration to airgapped environment

Now, we need to transfer the airgapped Ubuntu Pro configuration file, `server-ready.yml`, to the isolated Multipass instance. To do this, we have to transfer the file to the host machine and then to the isolated instance.

```sh
mulitpass transfer pro-configuration:server-ready.yml /tmp/server-ready.yml
multipass transfer /tmp/server-ready.yml livepatch-deploy:server-ready.yml
rm /tmp/server-ready.yml
```

### Step 4: Deploy airgapped Ubuntu Pro server

Now it is time to deploy the airgapped Ubuntu Pro server in the airgapped environment. To begin, we need an interactive shell in the isolated Multipass instance:

```sh
multipass shell livepatch-deploy
```

Next step is installing `contracts-airgapped` tool.

```sh
sudo add-apt-repository ppa:yellow/ua-airgapped
sudo apt update
sudo apt install contracts-airgapped
```

```{note}
In a real airgapped environment there will be no Internet access. So, one should use other methods, like local mirrors/packages, to install the dependencies via `apt` or `snap`. Setting up a fully isolated airgapped environment is out of the scope of this tutorial. So, we simply install dependencies from the Internet.
```

Once the installation is done, we need to run the airgapped Ubuntu Pro server with the configuration file we transferred to the instance in the previous step:

```sh
contracts-airgapped --input=./server-ready.yml
```

The airgapped Ubuntu Pro server is now listening on TCP port `8484`.

```{note}
This command runs the airgapped Ubuntu Pro server in the foreground. We still need to work on this Multipass instance. So, you can either open a new shell to the instance or run it in the background by appending a `&` to the command.
```

### Step 5: Deploy Livepatch on-prem server

We should now deploy Livepatch on-prem server in the airgapped environment. For simplicity, we will reuse the same Multipass instance we used for running the airgapped Ubuntu Pro server.

Livepatch on-prem requires a PostgreSQL database to work. Here, we use Docker Engine to spin up a PostgreSQL instance. Since the Multipass instance we are in does not have Docker, you need to install it by following the official [instructions](https://docs.docker.com/engine/install/ubuntu/). Once Docker Engine is installed, you can create a PostgreSQL container by using this command:

```sh
docker run \
  --name postgresql \
  -e POSTGRES_USER=livepatch \
  -e POSTGRES_PASSWORD=testing \
  -p 5432:5432 \
  -d postgres:12.11
```

```{note}
Livepatch on-prem server requires PostgreSQL 12 or above.
```

Now, we are ready to install Livepatch on-prem server:

```sh
sudo snap install canonical-livepatch-server
```

Before configuring Livepatch on-prem to communicate with our PostgreSQL database, we need to prepare the database:

```sh
canonical-livepatch-server.schema-tool postgresql://livepatch:testing@localhost:5432/livepatch
```

Once the database preparation is done, we can configure Livepatch on-prem database connection by the following command:

```sh
sudo snap set canonical-livepatch-server lp.database.connection-string=postgresql://livepatch:testing@localhost:5432/livepatch
```

Next, the Livepatch on-prem server should be configured to communicate with the airgapped Ubuntu Pro server:

```sh
sudo snap set canonical-livepatch-server \
  lp.contracts.enabled=true \
  lp.contracts.url=http://127.0.0.1:8484
```

Now, the Livepatch on-prem server is running and listening on TCP port `8080`. To test it, you can use `curl` like this:

```sh
curl http://localhost:8080
# Canonical Livepatch Health service, version v1.14.3
```

```{note}
By default, Livepatch on-prem server uses filesystem to stores the patches. The directory is located at `/var/snap/canonical-livepatch-server/common/patches`. So, in a real-world setup, you can download the latest patches by using the Patch Downloader tool, transfer them to the mentioned path, and use the Admin tool to refresh patch information. Check out [this](/server/how-to-guides/patch-management/use-the-patch-downloader-tool) topic on how to use the Patch Downloader tool.
```

### Step 7: Set up Livepatch client

In a real-world scenario, Livepatch clients run on different machines than those serving the Livepatch on-prem server. Since network configuration is out of the scope of this tutorial, we reuse the VM we have used so far, to install and configure the Livepatch client.

Before proceeding with the Livepatch client, we should first instruct the Ubuntu Pro client on the machine to communicate with the airgapped Ubuntu Pro server:

```sh
sudo sed -i -e 's|contract_url:.*|contract_url: http://127.0.0.1:8484|g' /etc/ubuntu-advantage/uaclient.conf
```

You should also instruct the Ubuntu Pro client to refresh its internal state for changes to take effect:

```sh
sudo pro refresh
```

More than that, we still need to map `livepatch.test.com` to the loopback interface IP address (i.e., `127.0.0.1`):

```sh
echo "127.0.0.1 livepatch.test.com" | sudo tee -a /etc/hosts
```

With Ubuntu Pro client being configured, we are ready to install the Livepatch client:

```sh
sudo snap install canonical-livepatch
```

By default, the Livepatch client is configured to communicate with the upstream Livepatch server. We need to change it so that the client speaks to our Livepatch on-prem server:

```sh
sudo canonical-livepatch config remote-server='http://livepatch.test.com'
```

Next, is to call `pro attach` and provide it with your Ubuntu Pro subscription token. You have already used the same token in an earlier step. Replace the `<TOKEN>` placeholder below with the same token and run the command:

```sh
sudo pro attach <TOKEN>
```

This might fail because we did not fully set up the airgapped Ubuntu Pro server (e.g., apt repository mirrors). But for our purposes, it is okay and we can continue with enabling Livepatch:

```sh
sudo pro enable livepatch
```

This should finish successfully. We can now check the status of the Livepatch client by running the following command:

```sh
sudo canonical-livepatch status
last check: 19 seconds ago
kernel: 5.15.0-119.129-generic
server check-in: succeeded
```

At this point, our Livepatch client is talking to our airgapped Livepatch on-prem server.

## Cleaning up

Since we used Multipass for this tutorial, we just need to delete the created instances:

```sh
multipass stop pro-configuration
multipass delete --purge pro-configuration
multipass stop livepatch-deploy
multipass delete --purge livepatch-deploy
```

## Summary

In this tutorial, we deployed an airgapped Livepatch on-prem server, alongside an Ubuntu Pro server enabling airgapped operations. Then, we configured the Ubuntu Pro client and Livepatch client to communicate with our airgapped servers.

