---
myst:
  html_meta:
    description: "Understand Livepatch Server security releases and update delivery for the machine charm, Kubernetes charm, and standalone snap."
---

(server-explanation-security-lifecycle)=

# Security lifecycle

> See also: {ref}`Security overview <server-explanation-security-overview>`, {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`

The Livepatch Server is released as a Kubernetes charm, a machine charm, and a standalone snap. Releases are made ad-hoc as features and bug fixes are implemented. When a security vulnerability in an upstream dependency is detected, a best-effort attempt is made to upgrade the dependency to the latest version that fixes the vulnerability. This security fix is then included in the next ad-hoc release.

A fix for a reported security issue that is directly present in the on-premises Livepatch Server can be prioritized and released as a minor version upgrade for the charms and snap.

Updates can apply database migrations automatically. Backups and recovery planning must account for the database as well as the application: reverting a binary does not necessarily restore compatibility with an upgraded schema. Independently deployed components, including the host operating system, database, TLS frontend, and Juju or Kubernetes infrastructure, have their own update lifecycles. See {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>` for update procedures.

## Charm

The [machine charm](https://charmhub.io/canonical-livepatch-server) and [Kubernetes charm](https://charmhub.io/canonical-livepatch-server-k8s) are published on Charmhub. Security updates are delivered as new charm revisions on the `latest/stable` channel. The machine charm selects a pinned server snap revision and holds its automatic refreshes, and a charm update selects the corresponding workload revision. Independently refreshing the managed snap or replacing the container image bypasses this version management.

Juju does not auto-refresh charms by default. Operators are responsible for periodically checking for updates and applying them.

## Standalone snap

The Livepatch Server snap is published on the [Snap Store](https://snapcraft.io/canonical-livepatch-server). Install the snap from the `latest/stable` channel.

By default, the snapd daemon checks for updates four times a day and applies them automatically. This default behavior can be modified using the `snap refresh` command to manually check for updates, postpone updates, schedule update windows, or roll back applied updates. For more information, see the [snap documentation on managing updates](https://snapcraft.io/docs/how-to-guides/manage-snaps/manage-updates/#refresh-update-control).

Modifying the default auto-update behavior could leave systems vulnerable to security issues addressed in newer releases. To verify that a security update has been applied, run `snap info canonical-livepatch-server` and confirm that the installed version matches the version in the `latest/stable` channel. If versions differ, manually apply the update:

```shell
sudo snap refresh canonical-livepatch-server --channel=latest/stable
```

## OCI image

Container image updates are delivered through new charm revisions that reference updated image digests.