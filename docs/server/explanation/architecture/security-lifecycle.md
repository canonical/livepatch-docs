---
myst:
  html_meta:
    description: "Understand Livepatch Server security releases and update delivery for the machine charm, Kubernetes charm, and standalone snap."
---

(server-explanation-security-lifecycle)=

# Security lifecycle

> See also: {ref}`Security overview <server-explanation-security-overview>`, {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>`, {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`

The Livepatch Server is released as a Kubernetes charm, a machine charm, and a standalone snap. Releases are made ad-hoc as features and bug fixes are implemented. When a security vulnerability in an upstream dependency is detected, a best-effort attempt is made to upgrade the dependency to the latest version that fixes the vulnerability. This security fix is then included in the next ad-hoc release.

A fix for a reported security issue that is directly present in the on-premises Livepatch Server can be prioritized and released as a minor version upgrade for the charms and snap.

See [Security support](/server/explanation/architecture/security-overview.md#security-support) for the stable-channel expectations, security-support policy, and contact information.

## Charm distributions

The [machine charm](https://charmhub.io/canonical-livepatch-server) and [Kubernetes charm](https://charmhub.io/canonical-livepatch-server-k8s) are published on Charmhub. Security updates are delivered as new charm revisions on `latest/stable`.

Each charm manages its workload version:

| Distribution | Workload update delivery |
| ------------ | ------------------------ |
| Machine charm | The charm selects a pinned server snap revision and holds its automatic refreshes. A charm update selects the corresponding workload revision. |
| Kubernetes charm | The charm revision references the workload OCI image. Image updates are delivered with charm updates. |

Juju does not automatically refresh these charms by default. Operators are responsible for reviewing releases and updating the charm, or arranging their own controlled automation. Independently refreshing the managed snap or replacing the container image bypasses this version management.

## Standalone snap

The [Livepatch Server snap](https://snapcraft.io/canonical-livepatch-server) is published on the Snap Store. Standalone deployments are expected to track `latest/stable`.

By default, snapd checks for updates four times a day and applies them automatically. An operator can control scheduling or hold automatic updates.

Postponing or disabling automatic updates can leave known vulnerabilities unpatched. Operators who hold updates remain responsible for checking releases and applying security fixes.

## Database and dependency updates

Updates can apply database migrations automatically. Backups and recovery planning must account for the database as well as the application: reverting a binary does not necessarily restore compatibility with an upgraded schema.

The server release includes its packaged application dependencies. Independently deployed components, including the host operating system, database, TLS frontend, and Juju or Kubernetes infrastructure, have their own update lifecycles.

See {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>` for manual update procedures, automatic-update controls, verification, release checks, and support.