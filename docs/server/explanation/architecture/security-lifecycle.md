---
myst:
  html_meta:
    description: "Understand the security lifecycle of the Livepatch on-premises server including update delivery for the charm, snap, and OCI image distribution channels."
---

(server-explanation-security-lifecycle)=
# Security lifecycle

> See also: {ref}`Security overview <server-explanation-security-overview>`, {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`

The on-premises Livepatch server is released as a Kubernetes charm and a snap. Releases are made ad-hoc as features and bug fixes are implemented. When a security vulnerability in an upstream dependency is detected, a best-effort attempt is made to upgrade the dependency to the latest version that fixes the vulnerability. This security fix is then included in the next ad-hoc release.

A fix for a reported security issue that is directly present in the on-premises Livepatch server can be prioritized and released as a minor version upgrade for both the charm and snap.

## Charm

The Livepatch charm (`canonical-livepatch-server-k8s`) is published on [Charmhub](https://charmhub.io/canonical-livepatch-server-k8s). Security updates are delivered as new charm revisions on the `latest/stable` channel. Container image updates are bundled with charm revisions -- each revision references specific OCI image versions.

To check for and apply updates:

```shell
juju refresh canonical-livepatch-server-k8s --channel=latest/stable
```

Juju does not auto-refresh charms by default. Operators are responsible for periodically checking for updates and applying them.

The charm also supports schema migrations via a Juju action. After upgrading, run schema migrations if required:

```shell
juju run canonical-livepatch-server-k8s/leader schema-upgrade
```

Schema migrations are leader-only operations executed in a separate container (`livepatch-schema-upgrade`) using the same database connection.

## Snap

The Livepatch server snap is published on the [Snap Store](https://snapcraft.io/canonical-livepatch-server). Install the snap from the `latest/stable` channel.

By default, the snapd daemon checks for updates four times a day and applies them automatically. This default behavior can be modified using the `snap refresh` command to manually check for updates, postpone updates, schedule update windows, or roll back applied updates. For more information, see the [snap documentation on managing updates](https://snapcraft.io/docs/how-to-guides/manage-snaps/manage-updates/#refresh-update-control).

Modifying the default auto-update behavior could leave systems vulnerable to security issues addressed in newer releases. To verify that a security update has been applied, run `snap info canonical-livepatch-server` and confirm that the installed version matches the version in the `latest/stable` channel. If versions differ, manually apply the update:

```shell
sudo snap refresh canonical-livepatch-server --channel=latest/stable
```

## OCI image

The Livepatch server OCI image is published for use in Kubernetes deployments managed by the charm. The image is built on a minimal `bare` base with the following properties:

- Compiled with `CGO_ENABLED=0` (no C library dependencies)
- Includes only `ca-certificates`, `logrotate`, `busybox-static`, and `base-files`

Container image updates are delivered through new charm revisions that reference updated image digests.