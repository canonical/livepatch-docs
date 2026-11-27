---
myst:
  html_meta:
    description: "Prepare a secure Livepatch Server configuration and distinguish configuration-only resets from destructive application and data resets."
---

(server-how-to-guides-configure-livepatch-server-securely)=

# Configure a Livepatch Server securely

This guide applies to the Kubernetes charm, machine charm, and standalone snap.

## Pre-deployment checklist

Before making the server available to production clients:

- Select `latest/stable` for your charm or standalone snap, check the [resource requirements](/server/reference/platform/resource-requirements.md), and restrict administrative access to the host and management infrastructure.
- Identify the PostgreSQL database and patch storage that belong to this deployment, and plan their access controls, credentials, capacity, and protection at rest. Do not use demonstration passwords from deployment examples.
- Choose your patch source: a configured upstream for connected synchronization, or {ref}`offline patch imports <server-how-to-guides-distribute-patches-offline>` for an air-gapped deployment.
- Plan network restrictions for client, administrator, monitoring, database, and storage traffic, and use TLS across untrusted networks.
- Prepare individual administrator accounts, the client auth-token workflow, and protected credential storage, using Juju secrets where supported. Decide whether to store machine reports, and review {ref}`upstream data exchange <server-reference-data-sent-to-canonical-servers>` for connected deployments.
- Arrange security logging, monitoring, backups, and an update process using the {ref}`logging and monitoring guide <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>` and {ref}`security lifecycle guidance <server-explanation-security-lifecycle>`. Test recovery before relying on backups.

## Apply the initial configuration

Manage charm configuration through Juju, and standalone snap configuration through the `lp.`-prefixed settings in the {ref}`configuration reference <server-reference-configuration>`.

1. Deploy using the [deployment tutorials](/server/tutorial/index.md) or the {ref}`standalone snap guide <server-how-to-guides-getting-started-with-the-livepatch-server-snap>`. Keep the server isolated from untrusted networks while you configure the database and {ref}`patch storage <server-reference-patch-storage>`.
2. Configure the ingress or reverse proxy, transport security, network restrictions, and request limits using the {ref}`hardening guide <server-how-to-guides-harden-your-livepatch-server-deployment>`.
3. Enable `auth.basic.enabled` with individual administrator accounts, as described in {ref}`administrator setup <server-how-to-guides-how-to-setup-livepatch-on-prem-server-administration-tool>`.
4. Configure logging, monitoring, backups, and [machine report](/server/reference/platform/configuration.md#machine-reports) retention.
5. Configure the upstream and synchronization credential, or use offline patch imports. Issue client auth tokens and {ref}`configure clients to use your on-premises server <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`.
6. Before allowing production traffic, test administrator access with valid and invalid credentials, client registration, and patch retrieval.

(server-how-to-guides-reset-livepatch-server-configuration)=

## Reset configuration without deleting data

A configuration-only reset restores selected settings and keeps the database and patch files. It does not erase stored reports or revoke client tokens.

```{warning}
Resetting authentication, listener, database, or storage settings can lock out administrators or disconnect the server from its data. Record the current settings first.
```

For a charm, reset only the options you intend to change, for example `juju config livepatch --reset server.log-level`. For a standalone snap, set each option to its default explicitly, for example `sudo snap set canonical-livepatch-server lp.server.log-level=info`. Afterwards, confirm administrator access, client requests, and patch retrieval.

(server-how-to-guides-fully-reset-livepatch-server)=

## Perform a full reset

```{warning}
A full reset is destructive. Removing the database invalidates all auth, machine, and sync tokens and deletes reports and patch metadata.
```

1. Follow {ref}`Decommission a Livepatch Server securely <server-how-to-guides-decommission-livepatch-server-securely>` to remove this deployment's application, database, patch storage, and snapshots. Redeploying the application alone is not enough if it reconnects to the existing database or patch store.
2. Redeploy from `latest/stable` with a fresh database and empty patch storage, and repeat the initial configuration above.
3. Recreate administrator credentials, issue new client auth tokens, register clients, and synchronize or import patches before restoring production traffic.

Backups, log archives, and exported reports are not erased by a reset; manage them under your retention policy.
