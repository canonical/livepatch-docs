---
myst:
  html_meta:
    description: "Complete a hands-on tutorial for airgapped Livepatch on-prem using Snaps. Deploy Livepatch and an airgapped Ubuntu Pro server in about 45 minutes."
---

(server-tutorial-airgapped-livepatch-and-snap)=

# Getting started with airgapped Livepatch on-prem and Snaps

> See also: {ref}`server`

This tutorial guides you through the process of deploying Livepatch on-prem in an airgapped environment using Snap packages. Livepatch on-prem enables the delivery of kernel patches to machines within network-restricted environments. For organisations with strict security requirements, deploying Livepatch on-prem in an airgapped setup ensures that the server operates without direct communication to the upstream Livepatch service.

In an airgapped environment, two additional tools replace the server's direct connection to Canonical's hosted Livepatch service:

- The **airgapped Ubuntu Pro server** provides Ubuntu Pro subscription services, including machine authentication and authorisation, without requiring Internet access.
- The **Patch Downloader** is a CLI tool for downloading the latest patch files from the upstream Livepatch Server. Administrators use this tool on an Internet-connected machine, then transfer the downloaded patches to the airgapped patch storage.

```{note}
When deploying airgapped Livepatch on-prem using Snaps, configure the patch storage to use an independently accessible option such as the filesystem or an S3/Swift bucket instead of PostgreSQL. This allows administrators to download patches with the Patch Downloader tool and transfer them independently to the patch storage.
```

Completing this tutorial should take approximately 45 minutes.

## Prerequisites

Before starting this tutorial, you'll need the following tools and resources.

### Ubuntu Pro token

You'll need an [Ubuntu Pro token](https://ubuntu.com/pro). Ubuntu Pro is free for up to five machines.

If you already have an Ubuntu Pro account, copy your token from the [Ubuntu Pro dashboard](https://ubuntu.com/pro/dashboard). If you don't have an account, sign up for a [free personal Ubuntu Pro account](https://ubuntu.com/pro), then copy your token.

### Multipass

Multipass is a CLI tool for launching Ubuntu VMs from Windows, Linux, and macOS.

Install Multipass from the Snap Store:

```bash
sudo snap install multipass
```

## Create the Multipass instances

This tutorial uses two Multipass VMs:

- `pro-configuration` -- an Internet-connected VM used to generate the airgapped Ubuntu Pro server configuration.
- `livepatch-deploy` -- the isolated VM representing your airgapped environment where the Livepatch on-prem server is deployed.

```bash
multipass launch jammy --name pro-configuration
multipass launch jammy --name livepatch-deploy -d 10G
```

## Generate the airgapped Ubuntu Pro server configuration

Open an interactive shell on the `pro-configuration` instance:

```bash
multipass shell pro-configuration
```

Install the `pro-airgapped` configuration tool:

```bash
sudo add-apt-repository ppa:yellow/ua-airgapped
sudo apt update
sudo apt install pro-airgapped
```

Create a configuration override file with your Ubuntu Pro token. Replace `<TOKEN>` with your token. Set the `remoteServer` value to the hostname you intend to use for your Livepatch on-prem server:

```bash
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
The hostname `livepatch.test.com` is used throughout this tutorial. You can substitute a different hostname, but remember to use it consistently in all subsequent steps.
```

The `pro-airgapped` tool requires Internet access to communicate with upstream Canonical services and fetch your subscription details. Run the following to generate the final configuration file (`server-ready.yml`):

```bash
cat override.yml | pro-airgapped > server-ready.yml
```

Exit the `pro-configuration` instance:

```bash
exit
```

## Transfer the configuration to the airgapped environment

Transfer the `server-ready.yml` file from `pro-configuration` to your host machine, then to the isolated `livepatch-deploy` instance:

```bash
multipass transfer pro-configuration:server-ready.yml /tmp/server-ready.yml
multipass transfer /tmp/server-ready.yml livepatch-deploy:server-ready.yml
rm /tmp/server-ready.yml
```

## Deploy the airgapped Ubuntu Pro server

Open an interactive shell on the `livepatch-deploy` instance:

```bash
multipass shell livepatch-deploy
```

Install the `contracts-airgapped` tool:

```bash
sudo add-apt-repository ppa:yellow/ua-airgapped
sudo apt update
sudo apt install contracts-airgapped
```

```{note}
In a real airgapped environment there is no Internet access. In that case, you must use local mirrors or packages to install dependencies via `apt` or `snap`. Setting up a fully isolated airgapped environment is outside the scope of this tutorial, so dependencies are installed directly from the Internet for simplicity.
```

Run the airgapped Ubuntu Pro server with the configuration file you transferred:

```bash
contracts-airgapped --input=./server-ready.yml
```

The server starts listening on TCP port `8484`. This command runs in the foreground. Open a second shell to the `livepatch-deploy` instance, or run it in the background by appending `&`:

```bash
contracts-airgapped --input=./server-ready.yml &
```

## Deploy Livepatch on-prem

Livepatch on-prem requires a PostgreSQL database. Install Docker Engine and create a PostgreSQL container. Follow the [Docker Engine installation instructions for Ubuntu](https://docs.docker.com/engine/install/ubuntu/), then run:

```bash
docker run \
  --name postgresql \
  -e POSTGRES_USER=livepatch \
  -e POSTGRES_PASSWORD=testing \
  -p 5432:5432 \
  -d postgres:12.11
```

```{note}
Livepatch on-prem requires PostgreSQL 12 or above.
```

Install the Livepatch on-prem server snap:

```bash
sudo snap install canonical-livepatch-server
```

Prepare the database schema:

```bash
canonical-livepatch-server.schema-tool postgresql://livepatch:testing@localhost:5432/livepatch
```

Configure the database connection:

```bash
sudo snap set canonical-livepatch-server lp.database.connection-string=postgresql://livepatch:testing@localhost:5432/livepatch
```

Configure Livepatch on-prem to communicate with the airgapped Ubuntu Pro server:

```bash
sudo snap set canonical-livepatch-server \
  lp.contracts.enabled=true \
  lp.contracts.url=http://127.0.0.1:8484
```

The Livepatch on-prem server is now running and listening on TCP port `8080`. Verify it is operational:

```bash
curl http://localhost:8080
# Canonical Livepatch Health service, version v1.14.3
```

## Set up the Livepatch Client

In a real-world scenario, Livepatch Clients run on separate machines. For this tutorial, you'll reuse the same VM.

Configure the Ubuntu Pro client to communicate with the airgapped Ubuntu Pro server:

```bash
sudo sed -i -e 's|contract_url:.*|contract_url: http://127.0.0.1:8484|g' /etc/ubuntu-advantage/uaclient.conf
```

Refresh the Ubuntu Pro client's internal state:

```bash
sudo pro refresh
```

Map the Livepatch on-prem hostname to the loopback address:

```bash
echo "127.0.0.1 livepatch.test.com" | sudo tee -a /etc/hosts
```

Install the Livepatch Client:

```bash
sudo snap install canonical-livepatch
```

Configure the Livepatch Client to communicate with your on-prem server instead of the upstream service:

```bash
sudo canonical-livepatch config remote-server='http://livepatch.test.com'
```

Attach your Ubuntu Pro subscription. Replace `<TOKEN>` with your Ubuntu Pro token:

```bash
sudo pro attach <TOKEN>
```

```{note}
`pro attach` may fail if the airgapped Ubuntu Pro server is not fully configured (for example, without apt repository mirrors). This is expected for the purposes of this tutorial.
```

Enable Livepatch:

```bash
sudo pro enable livepatch
```

Verify the Livepatch Client status:

```bash
sudo canonical-livepatch status
```

The output confirms that the client is communicating with your airgapped Livepatch on-prem server:

```text
last check: 19 seconds ago
kernel: 5.15.0-119.129-generic
server check-in: succeeded
```

## Managing patches in an airgapped environment

By default, Livepatch on-prem stores patches on the filesystem at `/var/snap/canonical-livepatch-server/common/patches`. To provide patches to the airgapped server, use the Patch Downloader tool on an Internet-connected machine to download the latest patches, transfer them to the patch storage path, then use the admin tool to refresh the patch information.

See the [Patch Downloader usage guide](/server/how-to-guides/patch-management/use-the-patch-downloader-tool) for instructions on downloading patches, and the [patch storage reference](/server/reference/patch-storage/index) for information on configuring alternative storage backends.

## Cleanup

Delete the Multipass VMs:

```bash
multipass stop pro-configuration
multipass delete --purge pro-configuration
multipass stop livepatch-deploy
multipass delete --purge livepatch-deploy
```

## Summary

In this tutorial, you deployed an airgapped Livepatch on-prem server alongside an airgapped Ubuntu Pro server using Snap packages and Docker, integrated the two services, and configured a Livepatch Client to communicate with the airgapped servers. The on-prem server now operates without direct communication to the upstream Livepatch service.

From here, you have several options:

- **Download and transfer patches**: Use the Patch Downloader tool to provide patches to your airgapped server. See the [Patch Downloader usage guide](/server/how-to-guides/patch-management/use-the-patch-downloader-tool).
- **Configure patch storage**: Set up an S3 or Swift bucket for patch storage in the airgapped environment. See the [patch storage reference](/server/reference/patch-storage/index).
- **Explore the MicroK8s deployment**: Deploy airgapped Livepatch on-prem on MicroK8s instead. See the [airgapped Livepatch and MicroK8s tutorial](/server/tutorial/airgapped-livepatch-and-microk8s).
- **Get support**: Canonical customers can receive support through the [Canonical support portal](https://portal.support.canonical.com/).