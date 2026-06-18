---
myst:
  html_meta:
    description: "Complete a hands-on tutorial for Livepatch on-prem on Kubernetes. Deploy Livepatch Server using MicroK8s and Juju, configure authentication, and sync patches in about 45 minutes."
---

(server-tutorial-getting-started-with-livepatch-on-prem-and-microk8s)=

# Getting started with Livepatch on-prem and MicroK8s

> See also: {ref}`server`

This tutorial guides you through the process of deploying Livepatch on-prem as a Kubernetes application using MicroK8s and Juju. You'll bootstrap a Juju controller on MicroK8s, deploy the Livepatch Server bundle, configure the URL template and ingress, set up authentication, sync patches, and verify that the server is ready to serve clients.

Completing this tutorial should take approximately 45 minutes.

## Prerequisites

Before starting this tutorial, you'll need the following tools and resources.

### Ubuntu Pro token

Livepatch on-prem requires a subscription token to authorise the on-prem instance to pull patch information from the upstream Livepatch service hosted by Canonical.

If you already have an Ubuntu Pro account, copy your token from the [Ubuntu Pro dashboard](https://ubuntu.com/pro/dashboard). If you don't have an account, sign up for a [free personal Ubuntu Pro account](https://ubuntu.com/pro), then copy your token.

### Multipass (optional)

Multipass is a CLI tool for launching Ubuntu VMs from Windows, Linux, and macOS. You can run the remainder of this tutorial from within a Multipass VM to avoid affecting your host machine.

Install Multipass from the snap store:

```bash
sudo snap install multipass
```

Launch a VM with sufficient resources:

```bash
multipass launch jammy --name livepatch-deploy --cpus 4 --memory 12G --disk 40G
```

Open an interactive shell into the VM:

```bash
multipass shell livepatch-deploy
```

## Install and configure MicroK8s

MicroK8s is a lightweight, CNCF-certified Kubernetes distribution. Install it from the snap store:

```bash
sudo snap install microk8s --channel=1.25-strict/stable
```

Add your user to the MicroK8s group and configure permissions:

```bash
sudo usermod -a -G snap_microk8s ubuntu
sudo chown -f -R ubuntu ~/.kube
newgrp snap_microk8s
```

Enable the required MicroK8s addons:

```bash
sudo microk8s enable hostpath-storage dns ingress
```

Create a short alias for the Kubernetes CLI:

```bash
sudo snap alias microk8s.kubectl kubectl
```

## Install and bootstrap Juju

Install Juju from the snap store:

```bash
sudo snap install juju --channel=3.1/stable
```

Create the local state directory and bootstrap a Juju controller on MicroK8s:

```bash
mkdir -p ~/.local/share
juju bootstrap microk8s livepatch-demo-controller
```

Create a model to host the Livepatch deployment:

```bash
juju add-model livepatch
```

## Deploy Livepatch on-prem

Deploy the Livepatch on-prem bundle from Charmhub. The bundle includes all the operators needed to run a working Livepatch on-prem server:

```bash
juju deploy canonical-livepatch-onprem --channel=k8s/stable --trust
```

Monitor the deployment progress:

```bash
juju status
```

After initialisation, the Livepatch unit will enter a blocked state with the message:

`"✘ patch-sync token not set, run get-resource-token action"`

### Provide the patch-sync token

Run the `get-resource-token` action with your Ubuntu Pro token to authorise the on-prem server. Replace `<token>` with your Ubuntu Pro token:

```bash
juju run livepatch/leader get-resource-token contract-token=<token> --wait 30s
```

A successful action returns output confirming that the token has been acquired.

### Configure the patch storage

Livepatch on-prem needs a place to store patches synced from the upstream Livepatch service. By default, the bundle stores patches in PostgreSQL. Other storage options -- including S3 -- are available and can be configured at deploy time. See the [patch storage reference](/server/reference/patch-storage/index) for more information.

## Configure the Livepatch server

### Set the URL template

The `server.url-template` option specifies the URL where Livepatch clients download patch files. The template must include the `{filename}` placeholder, which Livepatch replaces with the actual file name at runtime.

First, find the IP address of the Livepatch server pod. Run `juju status` and locate the IP address of the `livepatch` unit, or use the following command:

```bash
juju status --format json | jq -r '.applications.livepatch.units[]."address"'
```

Set the URL template using the pod's IP address. Replace `<ip-address>` with the value you retrieved:

```bash
juju config livepatch server.url-template="http://<ip-address>:8080/v1/patches/{filename}"
```

Verify the server is running:

```bash
curl <ip-address>:8080
# Canonical Livepatch Health service, version v1.13.1
```

### Configure the ingress

For production deployments, referencing pods by IP address is unreliable. Use a Kubernetes ingress to expose the Livepatch server through a stable domain name.

Configure the `service-hostname` on the nginx ingress integrator charm:

```bash
juju config ingress service-hostname=livepatch.test.com
```

After a short delay, verify the ingress was created:

```bash
kubectl get ingress -n livepatch
```

The output shows the IP address where the ingress is accessible with the `livepatch.test.com` domain name. Add an entry to your `/etc/hosts` file to resolve this domain locally:

```bash
echo '127.0.0.1 livepatch.test.com' | sudo tee -a /etc/hosts
```

Update the URL template to use the domain name:

```bash
juju config livepatch server.url-template="http://livepatch.test.com/v1/patches/{filename}"
```

Verify the server is accessible through the ingress:

```bash
curl livepatch.test.com
# Canonical Livepatch Health service, version v1.13.1
```

```{note}
To run this in a production environment, you must expose the MicroK8s cluster publicly.
```

### Deploy with a bundle overlay (optional)

You can apply these configuration options at deploy time using a Juju bundle overlay. Create an overlay file named `config.yaml`:

```yaml
applications:
  livepatch:
    options:
      url_template: <patch-url-template>
      external_hostname: <your-desired-hostname>
```

Deploy the bundle with the overlay:

```bash
juju deploy canonical-livepatch-onprem --channel=k8s/stable --overlay config.yaml
```

## Set up administrator authentication

Administrator access to the Livepatch on-prem deployment requires setting up basic authentication.

Install the `apache2-utils` package for `htpasswd`, which generates bcrypt password hashes:

```bash
sudo apt-get install apache2-utils
```

Generate a username and password hash pair. Replace `<username>` and `<password>` with your chosen credentials:

```bash
htpasswd -bnBC 10 <username> <password>
```

The output is a `username:hashed-password` pair. Use the output to configure Livepatch:

```bash
juju config livepatch auth.basic.enabled=true
juju config livepatch auth.basic.users='username:$2y$10$74ZgHaxn...UH/Bbw6W2bmctm'
```

See the [administration tool setup guide](/server/how-to-guides/security/setup-administration-tool) for instructions on installing the admin tool and setting up authentication.

### Log in with the admin tool

Install the administration tool from the snap store and create an alias:

```bash
sudo snap install canonical-livepatch-server-admin
sudo snap alias canonical-livepatch-server-admin.livepatch-admin livepatch-admin
```

Export the Livepatch server URL and log in:

```bash
export LIVEPATCH_URL="http://livepatch.test.com"
livepatch-admin login -a username:password
```

## Sync patches

Download patches from Canonical's hosted Livepatch server to your on-prem instance:

```bash
livepatch-admin sync trigger --wait
```

To verify synced patches:

```bash
livepatch-admin storage patches
```

To check for sync failures:

```bash
livepatch-admin sync report
```

To limit which patches are downloaded, see the [patch sync filters reference](/server/reference/patch-management/patch-sync-filters).

## Enable machine status reporting (optional)

Each Livepatch on-prem instance can optionally send information about the status of the machines it serves back to Canonical. This functionality is opt-in.

The information sent includes:

- Kernel version
- CPU model
- Architecture
- Boot time and uptime
- Livepatch client version
- Obfuscated machine ID
- Status of the patch currently applied to the machine's kernel

Enable machine status reporting:

```bash
juju config livepatch patch-sync.send-machine-reports=true
```

Disable reporting at any time by setting the value to `false`:

```bash
juju config livepatch patch-sync.send-machine-reports=false
```

## Cleanup

To clean up your Juju model and all associated storage:

```bash
juju destroy-model --no-prompt livepatch --destroy-storage
```

If you used Multipass, delete the VM:

```bash
multipass delete --purge livepatch-deploy
```

## Summary

In this tutorial, you deployed Livepatch on-prem on MicroK8s using Juju, configured an ingress for stable access, set up administrator authentication, and synchronised patches from the upstream Livepatch service. Your Livepatch on-prem server is now ready to serve clients.

From here, you have several options:

- **Set up Livepatch clients**: Install and configure the Livepatch client on machines in your infrastructure. See the [Livepatch Client documentation](/client/index).
- **Configure patch sync filters**: Limit which patches are downloaded to your on-prem server. See the [patch sync filters reference](/server/reference/patch-management/patch-sync-filters).
- **Explore alternative deployment options**: Deploy Livepatch on-prem on LXD. See the [Livepatch and LXD tutorial](/server/tutorial/livepatch-and-lxd).
- **Get support**: Canonical customers can receive support through the [Canonical support portal](https://portal.support.canonical.com/).