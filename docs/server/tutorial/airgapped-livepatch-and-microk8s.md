---
myst:
  html_meta:
    description: "Complete a hands-on tutorial for airgapped Livepatch on-prem on MicroK8s. Deploy Livepatch and an airgapped Ubuntu Pro server using Juju in about 60 minutes."
---

(server-tutorial-airgapped-livepatch-and-microk8s)=

# Getting started with airgapped Livepatch on-prem and MicroK8s

> See also: {ref}`server`

This tutorial guides you through the process of deploying Livepatch on-prem in an airgapped environment using MicroK8s and Juju. Livepatch on-prem enables the delivery of kernel patches to machines within network-restricted environments. For organisations with strict security requirements, deploying Livepatch on-prem in an airgapped setup ensures that the server operates without direct communication to the upstream Livepatch service.

In an airgapped environment, two additional tools replace the server's direct connection to Canonical's hosted Livepatch service:

- The **airgapped Ubuntu Pro server** provides Ubuntu Pro subscription services, including machine authentication and authorisation, without requiring Internet access.
- The **Patch Downloader** is a CLI tool for downloading the latest patch files from the upstream Livepatch server. Administrators use this tool on an Internet-connected machine, then transfer the downloaded patches to the airgapped patch storage.

```{note}
When deploying airgapped Livepatch on-prem on MicroK8s, configure the patch storage to use an independently accessible option such as an S3 or Swift bucket instead of PostgreSQL or the filesystem. This allows administrators to download patches with the Patch Downloader tool and transfer them to the storage independently.
```

Completing this tutorial should take approximately 60 minutes.

## Prerequisites

Before starting this tutorial, you'll need the following tools and resources.

### Ubuntu Pro token

You'll need an [Ubuntu Pro token](https://ubuntu.com/pro). Ubuntu Pro is free for up to five machines.

If you already have an Ubuntu Pro account, copy your token from the [Ubuntu Pro dashboard](https://ubuntu.com/pro/dashboard). If you don't have an account, sign up for a [free personal Ubuntu Pro account](https://ubuntu.com/pro), then copy your token.

### Multipass

Multipass is a CLI tool for launching Ubuntu VMs from Windows, Linux, and macOS.

Install Multipass from the snap store:

```bash
sudo snap install multipass
```

Launch a VM with sufficient resources for the Livepatch deployment:

```bash
multipass launch jammy --name livepatch-deploy --cpus 6 --memory 12G --disk 40G
```

Open an interactive shell into the VM:

```bash
multipass shell livepatch-deploy
```

All subsequent commands in this tutorial are run from within this VM.

## Install and configure MicroK8s

Install MicroK8s from the snap store:

```bash
sudo snap install microk8s --channel=1.30-strict/stable
```

Configure MicroK8s for the current user:

```bash
sudo usermod -a -G snap_microk8s $USER
mkdir -p ~/.kube
chmod 0700 ~/.kube
newgrp snap_microk8s
```

Enable the required addons:

```bash
sudo microk8s enable rbac
sudo microk8s enable hostpath-storage
sudo microk8s enable ingress
```

Verify MicroK8s is running:

```bash
microk8s status --wait-ready
```

## Install and bootstrap Juju

Install Juju from the snap store:

```bash
sudo snap install juju --channel=3.5/stable
```

Modify the MicroK8s client configuration file so the API address is accessible to Juju controllers:

```bash
microk8s.config > /tmp/microk8s.config
sudo cp /tmp/microk8s.config /var/snap/microk8s/current/credentials/client.config
sudo snap restart microk8s
```

Bootstrap two Juju controllers: one on MicroK8s for the Livepatch on-prem deployment, and one on LXD for the airgapped Ubuntu Pro server.

```bash
juju bootstrap localhost pro-demo-controller
juju bootstrap microk8s livepatch-demo-controller
```

Verify both controllers are available:

```bash
juju controllers
```

## Deploy Livepatch on-prem

Create a Juju model and deploy the Livepatch on-prem bundle:

```bash
juju add-model livepatch
juju deploy canonical-livepatch-onprem --channel=k8s/stable --trust
```

Monitor the deployment:

```bash
juju status --watch 5s
```

Once the status of the `livepatch` application changes to *"patch-sync token not set..."*, the deployment is initialised.

```{note}
By default, the charm assumes an ordinary (non-airgapped) deployment and prompts for a patch-sync token. Since this is an airgapped deployment, you can ignore this message and proceed with integrating the airgapped Ubuntu Pro server.
```

### Configure the ingress

Configure the nginx ingress integrator charm with a domain name for the Livepatch server:

```bash
juju config ingress service-hostname=livepatch.test.com
```

After a short delay, verify the ingress resource was created:

```bash
microk8s kubectl get ingress -n livepatch
```

The output shows the IP address where the ingress is accessible. Add a corresponding entry to the VM's `/etc/hosts` file:

```bash
echo "127.0.0.1 livepatch.test.com" | sudo tee -a /etc/hosts
```

```{note}
In a real-world deployment, the airgapped Ubuntu Pro server may be deployed on a separate node. Administrators should follow their own procedures to enable communication between the deployments.
```

## Deploy the airgapped Ubuntu Pro server

Switch to the LXD controller and create a new model:

```bash
juju switch pro-demo-controller
juju add-model pro
```

Deploy the `pro-airgapped-server` charm:

```bash
juju deploy pro-airgapped-server
```

Provide your Ubuntu Pro token. Replace `<TOKEN>` with your token:

```bash
juju config pro-airgapped-server pro-tokens=<TOKEN>
```

Configure the charm with your Livepatch on-prem address:

```bash
juju config pro-airgapped-server entitlements-url-map='{"livepatch": {"remoteServer": "http://livepatch.test.com"},"livepatch-onprem": {"remoteServer": "http://livepatch.test.com"}}'
```

Once the application status shows `active`, create a Juju application offer to allow the Livepatch on-prem deployment on the MicroK8s controller to integrate with it:

```bash
juju offer pro-airgapped-server:livepatch-server pro-offer
```

Note the IP address of the `pro-airgapped-server/0` unit from `juju status`. You'll need this later when configuring the Livepatch client:

```bash
juju status
```

## Integrate Livepatch on-prem with the airgapped Ubuntu Pro server

Switch back to the Livepatch model:

```bash
juju switch livepatch-demo-controller
```

Consume the application offer and integrate it with the Livepatch application:

```bash
juju consume pro-demo-controller:pro.pro-offer
juju integrate livepatch pro-offer
```

When the integration completes, the status message on the `livepatch` application changes to *"url-template not set"*. Configure the URL template:

```bash
juju config livepatch server.url-template="http://livepatch.test.com/v1/patches/{filename}"
```

```{note}
The `{filename}` placeholder must not be omitted or replaced. Livepatch replaces it with the actual file name at runtime.
```

After a short time, the Livepatch application status should change to `active`. Verify the server is running:

```bash
curl http://livepatch.test.com
# Canonical Livepatch Health service, version v1.14.3
```

Your Livepatch on-prem server is now operating in airgapped mode, with no communication to the upstream Livepatch server.

## Set up the Livepatch client

In a real-world scenario, Livepatch clients run on separate machines. For this tutorial, you'll reuse the same VM to install and configure a client.

Configure the Ubuntu Pro client to communicate with the airgapped Ubuntu Pro server. Replace `<IP>` with the address of the `pro-airgapped-server` unit you noted earlier:

```bash
sudo sed -i -e 's|contract_url:.*|contract_url: http://<IP>:8484|g' /etc/ubuntu-advantage/uaclient.conf
```

Refresh the Ubuntu Pro client's internal state:

```bash
sudo pro refresh
```

Install the Livepatch client:

```bash
sudo snap install canonical-livepatch
```

Configure the Livepatch client to communicate with your on-prem server instead of the upstream service:

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

Verify the Livepatch client status:

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

By default, Livepatch on-prem uses the filesystem to store patches at `/var/snap/canonical-livepatch-server/common/patches`. To provide patches to the airgapped server, use the Patch Downloader tool on an Internet-connected machine to download the latest patches, transfer them to the patch storage path, then use the admin tool to refresh the patch information.

See the [Patch Downloader usage guide](/server/how-to-guides/patch-management/use-the-patch-downloader-tool) for instructions on downloading patches, and the [patch storage reference](/server/reference/patch-storage/index) for information on configuring alternative storage backends.

## Cleanup

Delete the Multipass VM:

```bash
multipass stop livepatch-deploy
multipass delete --purge livepatch-deploy
```

## Summary

In this tutorial, you deployed an airgapped Livepatch on-prem server on MicroK8s alongside an airgapped Ubuntu Pro server, integrated the two services, and configured a Livepatch client to communicate with the airgapped servers. The on-prem server now operates without direct communication to the upstream Livepatch service.

From here, you have several options:

- **Download and transfer patches**: Use the Patch Downloader tool to provide patches to your airgapped server. See the [Patch Downloader usage guide](/server/how-to-guides/patch-management/use-the-patch-downloader-tool).
- **Configure patch storage**: Set up an S3 or Swift bucket for patch storage in the airgapped environment. See the [patch storage reference](/server/reference/patch-storage/index).
- **Explore the Snap-based deployment**: Deploy airgapped Livepatch on-prem using Snaps instead. See the [airgapped Livepatch and Snap tutorial](/server/tutorial/airgapped-livepatch-and-snap).
- **Get support**: Canonical customers can receive support through the [Canonical support portal](https://portal.support.canonical.com/).