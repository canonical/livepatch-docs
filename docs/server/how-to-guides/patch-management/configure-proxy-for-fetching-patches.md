---
myst:
  html_meta:
    description: "How to configure proxy for fetching patches with Livepatch on-prem."
---

(server-how-to-guides-use-a-proxy-when-fetching-patches)=

# Use a proxy when fetching patches

The Livepatch on-premises server can fetch patches through an HTTP proxy. The configuration steps vary depending on the deployment platform.

See the [patch-sync config](/server/reference/platform/configuration.md) for more details.

## Juju deployments (latest charms)

If Livepatch on-premises has been deployed using Juju, run the following Juju configuration command:

```bash
juju config livepatch \
    patch-sync.proxy.enabled=true \
    patch-sync.proxy.http=http://proxy.example.com \
    patch-sync.proxy.https=http://proxy.example.com
```

## Juju deployments (deprecated charm)

If Livepatch on-premises has been deployed using Juju with the older reactive charm (see the [migration guide](/server/how-to-guides/deployment/migrate-from-reactive-charm-to-operator-charm.md)), run the following Juju configuration command:

```bash
juju config livepatch \
    http_proxy=http://proxy.example.com \
    https_proxy=http://proxy.example.com
```

## Snap deployments

If Livepatch on-premises has been deployed using Snap, run the following commands to configure a proxy:

```bash
sudo snap set canonical-livepatch-server lp.patch-sync.proxy.enabled=true
sudo snap set canonical-livepatch-server lp.patch-sync.proxy.http=http://proxy.example.com
sudo snap set canonical-livepatch-server lp.patch-sync.proxy.https=http://proxy.example.com
```

The applied configuration can be viewed by running the following:

```bash
sudo snap get canonical-livepatch-server lp.patch-sync.proxy
Key                          Value
lp.patch-sync.proxy.enabled  true
lp.patch-sync.proxy.http     http://proxy.example.com
lp.patch-sync.proxy.https    http://proxy.example.com
```