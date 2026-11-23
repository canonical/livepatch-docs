---
myst:
  html_meta:
    description: "Prepare a secure Livepatch Server configuration and distinguish configuration-only resets from destructive application and data resets."
---

(server-how-to-guides-configure-livepatch-server-securely)=

# Configure a Livepatch Server securely

Use this guide to prepare a secure configuration or reset an existing deployment. It applies to the Kubernetes charm, machine charm, and standalone snap.

For charmed deployments, manage configuration through Juju. For a standalone snap, use the `lp.`-prefixed settings described in the {ref}`configuration reference <server-reference-configuration>`. In the Juju examples below, the application is named `livepatch`; replace it with your application's name.

## Pre-deployment checklist

Before making the server available to production clients:

- Select `latest/stable` for your charm or standalone snap, check the [resource requirements](/server/reference/platform/resource-requirements.md), and restrict administrative access to the host and management infrastructure.
- Identify the PostgreSQL database and patch storage that belong to this deployment. Plan their access controls, credentials, capacity, and protection at rest. Do not use demonstration passwords from deployment examples.
- Choose your patch source: a configured upstream for connected synchronization, or {ref}`offline patch imports <server-how-to-guides-distribute-patches-offline>` for an air-gapped deployment.
- Plan network restrictions for client, administrator, monitoring, database, and storage traffic. Use TLS across untrusted networks; TLS remains optional within a trusted internal network.
- Prepare individual administrator accounts, the local client auth-token workflow, and protected credential storage. Use Juju secrets where supported and restrict access to snap configuration and administrator token files.
- Decide whether to store machine reports and configure their retention. Treat detailed upstream report forwarding as a separate decision; review {ref}`upstream data exchange <server-reference-data-sent-to-canonical-servers>` for connected deployments.
- Arrange security log collection, monitoring, and an update process using the {ref}`logging and monitoring guide <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>` and {ref}`security lifecycle guidance <server-explanation-security-lifecycle>`.
- Define backup, restore, retention, and deletion procedures for the database, patch storage, configuration, and exported data. Protect backups and test recovery before relying on them.

## Apply the initial configuration

1. Deploy your selected distribution using the [deployment tutorials](/server/tutorial/index.md) or {ref}`standalone snap guide <server-how-to-guides-getting-started-with-the-livepatch-server-snap>`. Keep it isolated from untrusted networks while configuring the database and {ref}`patch storage <server-reference-patch-storage>`.
2. Configure the intended ingress or reverse proxy and network restrictions before transmitting credentials across untrusted networks. Restrict direct access to the backend listener so clients cannot bypass proxy controls. Follow the {ref}`hardening guide <server-how-to-guides-harden-your-livepatch-server-deployment>` for transport security, network restrictions, and request limits.
3. Configure administrator accounts with bcrypt-hashed passwords using `auth.basic.users` or a supported secret-backed option, and enable `auth.basic.enabled`. Protect the credentials and use individual administrator identities. See the {ref}`authentication reference <server-reference-livepatch-server-authentication>` for the authentication mechanisms.
4. Configure logging, monitoring, backups, and your chosen machine-report retention settings. Local report storage is disabled by default; enabling it does not enable detailed upstream report forwarding. See the [machine reports configuration options](/server/reference/platform/configuration.md#machine-reports).
5. Configure the intended upstream and synchronization credential for a connected deployment, or use offline patch imports and internal dependencies for an air-gapped deployment. Issue client auth tokens for the required tiers and {ref}`configure clients to use your on-premises server <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`.
6. Before allowing production traffic, check administrator access with valid and invalid credentials, client registration and tier access with a test client, and patch retrieval from the intended storage. Confirm that network restrictions, logging, and the configured upstream or offline workflow operate as intended.

(server-how-to-guides-reset-livepatch-server-configuration)=

## Reset configuration without deleting data

A configuration-only reset restores selected settings while retaining the PostgreSQL database and patch files. It does not erase stored reports or revoke client tokens.

```{warning}
Configuration changes can restart the server or disrupt access. Resetting authentication, listener, database, or storage settings can lock out administrators or disconnect the server from its data. Record the current settings in a protected location and plan how to restore access before changing them.
```

### Kubernetes and machine charms

Use Juju to inspect the configuration and reset only the options you intend to change. Do not change the machine charm's managed snap or the Kubernetes workload environment directly.

For example, restore the charm's default log level:

```shell
juju config livepatch --reset server.log-level
juju config livepatch server.log-level
```

The default for this option is `info`. Repeat the per-option reset only for settings you intend to restore to the charm's defaults. Leave database relations and patch storage intact.

### Standalone snap

Restore each selected setting to its intended default explicitly. For example, restore the default log level:

```shell
sudo snap set canonical-livepatch-server lp.server.log-level=info
sudo snap get canonical-livepatch-server lp.server.log-level
```

Do not use empty values as a general reset mechanism. Preserve the deployment-specific database connection, storage settings, and credentials unless you intend to reconfigure them.

After either procedure, confirm administrator access, client requests, database connectivity, and patch retrieval before restoring normal traffic.

(server-how-to-guides-fully-reset-livepatch-server)=

## Perform a full reset

A full reset removes the application's configuration, database records, and patch data before creating a fresh deployment. Removing and redeploying the application alone is not sufficient if it reconnects to an existing database or patch store.

```{warning}
This procedure is destructive and interrupts patch delivery. Removing the database invalidates the auth, machine, and sync tokens stored in it and deletes reports and patch metadata. Do not destroy a shared Juju model, PostgreSQL service, or storage resource when removing this deployment's data.
```

1. Schedule downtime and stop the Livepatch application from accepting requests. Identify its actual database, patch storage locations, configuration, credentials, and snapshots before removing anything.
2. Decide which backups or exported data your retention policy permits you to retain. Protect retained copies separately; do not restore an old database or configuration backup if the goal is a clean state.
3. Follow the relevant application, database, patch storage, and snapshot removal procedures in {ref}`Decommission a Livepatch Server securely <server-how-to-guides-decommission-livepatch-server-securely>`. Remove only this deployment's data at its configured locations, including external PostgreSQL and patch storage. Do not assume that removing a charm or snap deletes external data.
4. Deploy the chosen charm or standalone snap from `latest/stable`, using a fresh database and empty patch storage. Follow the deployment instructions to initialize the schema and repeat the initial security configuration above.
5. Recreate administrator credentials, issue fresh client auth tokens, and register clients with the new deployment. Synchronize or import patches and check access controls and patch retrieval before restoring production traffic.

External log archives, backups, exported reports, and credential copies are not automatically erased by a reset. Manage them under your retention and deletion policies, and revoke or rotate external credentials as required. See the {ref}`security overview <server-explanation-security-overview>` for data lifecycle responsibilities.
