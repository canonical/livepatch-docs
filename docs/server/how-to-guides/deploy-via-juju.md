---
myst:
  html_meta:
    description: "How to deploy via juju with Livepatch on-prem."
---

(livepatch_on_prem-how-to-guides-how-to-deploy-livepatch-on-prem)=

# How to deploy Livepatch on-prem

## Prerequisites

```{warning}
This guide has been deprecated. Refer to our [tutorials](/server/tutorial/index.md) or [snap how-to](/server/how-to-guides/deploy-via-snap.md).

If you have an existing deployment using this document, see our [migration guide](/server/how-to-guides/migrate-from-reactive-charm-to-operator-charm.md) to update your deployment.
```

We will deploy and configure the livepatch on-prem server using Juju and Charmed Operators. Juju is an Open Source Charmed Operator Framework that controls the whole lifecycle of an application - including machine applications. Please follow the [installation instructions](https://documentation.ubuntu.com/juju/latest/howto/manage-juju/#install-juju) for your system.

The livepatch on-prem bundle needs to be deployed on machines running Ubuntu focal.

You don’t need to have previous or advanced knowledge of Juju or Charmed Operators to follow this guide and deploy livepatch.

## Livepatch authorization token

Since on-prem livepatch servers act as caching proxies for the livepatch service hosted by canonical, the subscription token is required to authorize the on-prem instance to pull patch information.

To get your Ubuntu Pro subscription token, please go to https://ubuntu.com/pro.
![image1|690x373](/_static/images/9IhZmAFso5ufkbB8QyboWUoROad.png)

## Deployment Steps

## 1. Initialize juju

Once you have Juju CLI installed, you will need to bootstrap a Juju controller to your cloud. The [Juju documentation](https://documentation.ubuntu.com/juju/latest/howto/manage-controllers/) has detailed instructions on how to do that for several clouds and machine types.

See[ resources topic ](/server/reference/resource-requirements.md) for requirements for the virtual machines running livepatch on-premises services.

## 2. Deploying the bundle

The bundle and charmed operators necessary to deploy livepatch server are available in the charmstore at

https://charmhub.io/canonical-livepatch-onprem

To start the deployment on a created juju model, run:

```
juju deploy ch:canonical-livepatch-onprem
```

## 3. Configuring livepatch

After the deployment completes, verify the status of the model by running:

```
juju status
```

The output should look like:
![screenshot_20210601_164245|648x159](/_static/images/z8RLGfl4BdhI501OBguUgqlpaKY.png)

At this point the livepatch unit is expected to be in a blocked state with the message:
"✘ sync_token not set"

Provide the token (acquired by following instructions in the Livepatch authorization token section) by running:

```
juju config ubuntu-advantage token="<token>"

juju run-action livepatch/leader get-resource-token --wait
```

The output should indicate the token has successfully been acquired:
![screenshot_20210601_164500|690x127](/_static/images/fboev1KPeCS41ZI0OUD6Zsde1vC.png)

After that, provide the url_template setting as follows:

```
juju config livepatch url_template="http://10.94.227.82/v1/patches/{filename}"
```

The url_template specifies the url where patch files can be downloaded by livepatch client agents. The url template should be of the form 'http(s)://{HOSTNAME}/v1/patches/{filename}'. The hostname is the only part that needs to be changed. The hostname can be just the ip address of the haproxy unit. If a DNS hostname is configured for the haproxy IP address, that can be used too.

### Deploying with a config overlay (Optional)

These settings can be configured at deploy-time by using a juju bundle overlay:

```
juju deploy ch:canonical-livepatch-onprem --overlay config.yaml
```

The overlay file should have the following content:

```
applications:
  livepatch:
    options:
      url_template: <patch url template>
  ubuntu-advantage:
    options:
      token: <contract token>
```

## 4. Setting up authentication

To enable admin tool access to the livepatch server, authentication needs to be configured. The easiest way is to enable username/password authentication.

Generate the password hash using:

```
sudo apt-get install apache2-utils

htpasswd -bnBC 10 <username> <password>
username:$2y$10$74ZpDgHaxnUQo.AJZk1cMuSRfef5oK5xq5o/GLbUH/Bbw6W2bmctm
```

Use the output of the previous command to configure livepatch:

```
juju config livepatch auth_basic_users='username:$2y$10$74ZgHaxn...UH/Bbw6W2bmctm'
```

See [Administration Tool](/server/how-to-guides/setup-administration-tool.md) topic for instructions on installing the administration tool and setting up authentication.

Once this has been done, the livepatch admin tool can be used to authenticate:

```
export LIVEPATCH_URL=http(s)://{haproxy url}

livepatch-admin login -a [username:password]
```

## 5. Downloading patches

The final step before attaching client machines to the server is to download patches from Canonical servers. This can be done using the admin tool. See [How to setup administration tool](/server/how-to-guides/setup-administration-tool.md) for installation steps.

To download patches, run:

```
livepatch-admin sync trigger --wait
```

## Enabling machine status reporting

Each livepatch on-prem instance can optionally send information about the status of the machines it's serving back to Canonical. This functionality is opt-in.

The information sent back about each machine includes:

- Kernel version
- CPU model
- Architecture
- Boot time and uptime
- Livepatch client version
- Obfuscated machine id
- Status of the patch currently applied to the machine's kernel

To enable this reporting, run the following juju command:

```
juju config livepatch sync_send_machine_reports=true
```

This can be disabled at any time by setting the flag to `false`.
