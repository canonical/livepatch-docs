---
myst:
  html_meta:
    description: "How to chain livepatch servers with Livepatch on-prem."
---


(server-how-to-guides-chain-on-prem-servers)=

# Chain On-Prem Servers

As your Livepatch deployment grows, you may want to scale out beyond a single deployment.
This how-to document shows you how to chain multiple Livepatch on-prem servers together.

This is useful in a few scenarios:

- Running the server in multiple availability zones or regions.
- Isolating machines to a specific on-prem instance.
- Increasing the scale of a deployment without HA.

In order to chain on-prem server you need:

- An existing Livepatch on-prem server.
- A new deployment of the Livepatch on-prem server.
- The Livepatch admin tool [setup](/server/how-to-guides/security/setup-administration-tool.md) against the existing server.

When deploying an instance of the Livepatch server (see the existing [guides](/server/how-to-guides/index.md)) you will normally use an Ubuntu Pro token to configure the on-premise server to sync with Canonical's hosted Livepatch server.

In this scenario, skip that step and use a token provided by the existing Livepatch server.

Below, the "upstream" server is referred to as the original Livepatch server and the "downstream" server as the new deployment that will sync from the "upstream".

## On the upstream server

Using the admin tool:

```
livepatch-admin sync-tokens add edge my-token
```

This will return a token that another on-prem server can use to sync patches.
Sync tokens, by default, do not expire but the `--valid-until` flag can be used to set an expiry date.

Change `edge` if you would like to sync from a different tier.

```{note}

See the [reference doc](/server/reference/patch-management/patch-management.md) to better understand how tiers work and how patches are synced.

```

## On the downstream server

Configure the following values on your charm/snap deployment:

- Set `patch-sync.token` using the previously obtained token.
- Set `patch-sync.upstream-url` to the upstream server URL.

You are now done configuring your downstream Livepatch server to sync from your upstream server.
This process can be repeated as many times as desired.

## Syncing tiers (Optional)

Conventionally, each Livepatch on-prem server provides its own patch management using local tiers.
Patches can be promoted between tiers to provide phased rollouts.

To cater for scenarios where patch management should only be done in one place and synced to various downstream servers, the ability to sync tiers is offered.

Enabling this feature will do the following:

- Match a server's tier structure to its upstream. This is a **destructive** operation and will remove any tiers that currently exist.

- Sync patch tier info from the upstream server. A patch e.g. 100.1 promoted to the "stable" tier upstream will automatically be reflected in the downstream server.

- Disables local patch management i.e. patches cannot be promoted between tiers and tiers cannot be created or destroyed through the admin tool.

This feature only needs to be enabled on the downstream server. No changes are needed on the upstream server.

Enable this functionality by setting the `patch-sync.sync-tiers` config value to `true`.
