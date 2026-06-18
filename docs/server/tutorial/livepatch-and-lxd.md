---
myst:
  html_meta:
    description: "Complete a hands-on tutorial for Livepatch on-prem. Deploy Livepatch Server using LXD and Juju, configure authentication, and sync patches in about 30 minutes."
---

(server-tutorial-livepatch-and-lxd)=

# Getting started with Livepatch on-prem and LXD

> See also: {ref}`server`

This tutorial guides you through the process of deploying Livepatch on-prem using LXD as your cloud provider. You'll bootstrap a Juju controller, deploy the Livepatch Server bundle, enable Ubuntu Pro, configure authentication, sync patches, and verify that the server is ready to serve clients.

Completing this tutorial should take approximately 30 minutes.

## Prerequisites

Before starting this tutorial, you'll need the following tools installed on your host machine.

### LXD

LXD provides a unified experience for managing system containers and virtual machines. Juju uses LXD to spawn containers for the Livepatch on-prem services.

Install LXD from the snap store:

```bash
sudo snap install lxd --channel=5.0/stable
```

Initialise LXD using the `--auto` flag to accept the defaults:

```bash
lxd init --auto
```

### Juju

Juju is an open source orchestration engine for software operators that enables the deployment, integration, and lifecycle management of applications at any scale, on any infrastructure.

Install Juju from the snap store:

```bash
sudo snap install juju
```

### JQ

JQ is a lightweight JSON processor. You'll use it to extract values from Juju's output during this tutorial.

```bash
sudo apt update && sudo apt install jq
```

### Ubuntu Pro token

Livepatch on-prem requires authorisation to the upstream Livepatch service hosted by Canonical. You'll need an [Ubuntu Pro token](https://ubuntu.com/pro) to enable Livepatch. Ubuntu Pro is free for up to five machines.

If you already have an Ubuntu Pro account, copy your token from the [Ubuntu Pro dashboard](https://ubuntu.com/pro/dashboard). If you don't have an account, sign up for a [free personal Ubuntu Pro account](https://ubuntu.com/pro), then copy your token.

## Bootstrap Juju

Create a Juju controller on LXD:

```bash
mkdir -p ~/.local/share
juju bootstrap lxd livepatch-onprem
```

The bootstrap operation takes a few moments to complete. Once finished, create a model to host the Livepatch deployment:

```bash
juju add-model livepatch
```

Verify that you're working in the correct model:

```bash
juju switch livepatch
```

## Deploy Livepatch on-prem

Deploy the Livepatch on-prem bundle from Charmhub:

```bash
juju deploy canonical-livepatch-onprem --channel=machine
```

Monitor the deployment progress with:

```bash
juju status --watch 2s
```

After some time, the model status will show all applications initialising and eventually settling into a stable state.

> See also: If you're migrating from the reactive charm to the operator charm, refer to the [migration guide](/server/how-to-guides/deployment/migrate-from-reactive-charm-to-operator-charm).

## Enable Ubuntu Pro (optional)

Enable Ubuntu Pro on the deployed machines for Expanded Security Maintenance (ESM). Replace `<token>` with your Ubuntu Pro token:

```bash
juju config ubuntu-advantage token='<token>'
```

On a successful attach, the status output will reflect the change.

If you're not using Ubuntu Pro, remove the `ubuntu-advantage` application:

```bash
juju remove-application ubuntu-advantage
```

## Enable Livepatch

Enable Livepatch by providing your Ubuntu Pro token to the Livepatch server unit:

```bash
juju run livepatch/0 enable token='<token>'
```

A successful action returns an output confirming that Livepatch is enabled.

## Configure the Livepatch server

### Set the URL template

The `server.url-template` option specifies the URL where Livepatch clients download patch files. The template must include the `{filename}` placeholder, which Livepatch replaces with the actual file name at runtime:

```
http(s)://domain/{filename}
```

For this tutorial, you'll use the Livepatch server itself to serve patches. The server exposes a dedicated endpoint at:

```
/v1/patches/:patch_name
```

The bundle includes HAProxy, which acts as a load balancer and reverse proxy. Use the HAProxy unit's address to construct the URL template. Run the following to set it automatically:

```bash
HAPROXY_ADDRESS=$(juju status --format json | jq -r '.applications.haproxy.units["haproxy/0"]["public-address"]') && echo $HAPROXY_ADDRESS
juju config livepatch server.url-template="http://$HAPROXY_ADDRESS/v1/patches/{filename}"
```

Confirm the configuration was applied:

```bash
juju config livepatch server.url-template
```

```{note}
For production deployments, you may use an AWS S3 bucket or another file server for patch storage. In that case, your URL template might resemble `https://s3-eu-west-2.amazonaws.com/livepatch/patches/{filename}`.
```

### Run the database schema migration

Trigger a database schema migration on the Livepatch server unit:

```bash
juju run livepatch/0 schema-upgrade
```

This operation only needs to be run once. Future upgrades will apply schema migrations automatically. Once completed, the Livepatch application will enter a running state.

## Set up administrator authentication

Administrator access to the Livepatch on-prem deployment requires setting up basic authentication.

Enable basic authentication on the Livepatch application:

```bash
juju config livepatch auth.basic.enabled=true
```

Install the `apache2-utils` package for `htpasswd`, which generates bcrypt password hashes:

```bash
sudo apt-get install apache2-utils -y
```

Generate a username and password hash pair. Replace `admin` and `admin123` with your chosen credentials:

```bash
htpasswd -bnBC 10 admin admin123
```

The output is a `username:hashed-password` pair. Use the output verbatim to configure Livepatch. Wrap the value in single quotes to escape special characters:

```bash
juju config livepatch auth.basic.users='admin:$2y$10$...'
```

To add additional administrators, provide a comma-separated list of `user:password` pairs.

## Configure the admin tool

The Livepatch administration tool allows you to manage the server from the command line. Install it from the snap store:

```bash
sudo snap install canonical-livepatch-server-admin
```

Create a convenient alias:

```bash
sudo snap alias canonical-livepatch-server-admin.livepatch-admin livepatch-admin
```

Export the Livepatch server URL (pointing to the HAProxy address you retrieved earlier):

```bash
export LIVEPATCH_URL="http://$HAPROXY_ADDRESS"
```

Log in with one of your administrator credentials:

```bash
livepatch-admin login -a admin:admin123
```

## Sync patches

Download patches from Canonical's hosted Livepatch server to your on-prem instance:

```bash
livepatch-admin sync trigger --wait
```

For more information on the admin tool, see the [administration tool setup guide](/server/how-to-guides/security/setup-administration-tool). To limit which patches are downloaded, see the [patch sync filters reference](/server/reference/patch-management/patch-sync-filters).

## Enable machine status reporting (optional)

Each Livepatch on-prem instance can optionally send information about the status of the machines it serves back to Canonical. Full details on the data that is transmitted are available in the [data sent reference](/client/reference/networking/data-sent).

Enable machine status reporting:

```bash
juju config livepatch patch-sync.send-machine-reports=true
```

Disable reporting at any time by setting the value to `false`:

```bash
juju config livepatch patch-sync.send-machine-reports=false
```

## Cleanup

When you're finished exploring Livepatch on-prem, destroy the Juju controller and all associated models:

```bash
juju destroy-controller livepatch-onprem --destroy-all-models
```

## Summary

In this tutorial, you deployed Livepatch on-prem using LXD and Juju, configured the server to serve patches, set up administrator authentication, and synchronised patches from the upstream Livepatch service. Your Livepatch on-prem server is now ready to serve clients.

From here, you have several options:

- **Set up Livepatch clients**: Install and configure the Livepatch client on machines in your infrastructure. See the [Livepatch Client documentation](/client/index).
- **Configure patch sync filters**: Limit which patches are downloaded to your on-prem server. See the [patch sync filters reference](/server/reference/patch-management/patch-sync-filters).
- **Explore other deployment options**: Deploy Livepatch on-prem on MicroK8s instead. See the [Livepatch and MicroK8s tutorial](/server/tutorial/livepatch-and-microk8s).
- **Get support**: Canonical customers can receive support through the [Canonical support portal](https://portal.support.canonical.com/).