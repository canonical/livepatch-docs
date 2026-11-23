---
myst:
  html_meta:
    description: "Check for and apply Livepatch Server security updates, manage standalone snap automatic updates, and obtain support or report vulnerabilities."
---

(server-how-to-guides-get-security-updates-and-support)=

# Get security updates and support

Keep the Livepatch Server on the current `latest/stable` charm revision or standalone snap revision. This guide covers updates to the server application, not the kernel patches that it distributes to clients. Synchronizing or importing kernel patches does not update the server itself.

The Juju examples assume that the Livepatch application is named `livepatch`. Replace it with your application name.

## Prepare for an update

Review the [server release notes](/release-notes/server/index.md) and any release-specific upgrade instructions before applying an update. Plan for service restarts and possible database migrations.

Take protected backups of the database, deployment configuration, and any patch data required by your recovery plan. Confirm that you can restore them. Restrict access to backups because they can contain credentials and fleet data.

Arrange a maintenance window appropriate to your availability requirements. Updates can apply database migrations automatically; reverting a charm or snap alone might not restore a compatible database.

The store-based procedures below require access to the relevant artifact stores and container registries through your approved network path. For a deployment without store access, arrange delivery of the required server artifacts through your offline deployment process before the maintenance window. Offline kernel-patch imports are not a substitute for server updates.

## Update a Juju-managed deployment

Use this procedure for both the machine charm and the Kubernetes charm.

Check the deployed charm revision and channel:

```shell
juju status livepatch
```

Compare the revision with the `latest/stable` channel on the appropriate Charmhub page:

* [Machine charm](https://charmhub.io/canonical-livepatch-server)
* [Kubernetes charm](https://charmhub.io/canonical-livepatch-server-k8s)

Apply the current stable revision:

```shell
juju refresh livepatch --channel=latest/stable
```

Allow the charm to complete its update and inspect the application and unit status again. Follow any release-specific instructions; do not routinely run an additional manual schema-upgrade action after a successful managed update.

Juju does not automatically refresh these charms by default. Schedule regular checks and operator-managed updates. If your organization automates charm refreshes, manage approval, postponement, and suspension in that automation rather than using snapd controls.

The machine charm manages a pinned server snap revision and holds its automatic refreshes. Do not manually refresh or unhold that snap. Update the charm so that it selects the corresponding workload revision. Similarly, update the Kubernetes charm rather than replacing its workload image independently.

## Update a standalone snap manually

This procedure is not for the snap managed by the machine charm.

Check the installed revision, tracked channel, and available stable revision:

```shell
snap list canonical-livepatch-server
snap info canonical-livepatch-server
```

Apply the current stable revision:

```shell
sudo snap refresh canonical-livepatch-server --channel=latest/stable
```

An explicit refresh naming the snap can update it even when its automatic refreshes are held. After updating, check the installed revision again and verify operation as described below.

## Manage standalone snap automatic updates

By default, snapd checks for updates four times a day and applies them automatically. Holds, refresh schedules, and store availability can change this behavior.

Inspect the refresh schedule and server snap:

```shell
snap refresh --time
snap list canonical-livepatch-server
```

### Enable or resume automatic updates

Ensure that the standalone snap tracks `latest/stable`, using the manual update procedure above if a channel change is needed. Remove its automatic-refresh hold:

```shell
sudo snap refresh --unhold canonical-livepatch-server
```

Check the refresh schedule again. Removing a per-snap hold does not remove other system-wide refresh restrictions or restore unavailable store access.

### Schedule update windows

For example, to allow refreshes between 02:00 and 04:00:

```shell
sudo snap set system refresh.timer=02:00-04:00
```

This setting affects **all snaps on the host**, not just Livepatch Server. Coordinate it with other services and existing maintenance policies. Confirm the resulting schedule with `snap refresh --time`.

### Postpone an automatic update

To hold automatic refreshes for 24 hours:

```shell
sudo snap refresh --hold=24h canonical-livepatch-server
```

Automatic refreshes become eligible again after the hold expires, subject to the refresh schedule and other restrictions. Set a deadline to review and apply the update; a hold can leave known vulnerabilities unpatched.

### Disable automatic updates

To hold automatic refreshes indefinitely:

```shell
sudo snap refresh --hold=forever canonical-livepatch-server
```

Use an indefinite hold only with an operator-managed update process. Assign responsibility for checking releases and applying fixes, and remove the hold with `--unhold` when automatic updates should resume. Disabling automatic updates does not remove the need for security updates.

See the [snap update-control documentation](https://snapcraft.io/docs/how-to-guides/manage-snaps/manage-updates/#refresh-update-control) for further scheduling and hold behavior.

## Verify the update

Check that the expected charm or standalone snap revision is installed and that the application is healthy. Confirm that an administrator can authenticate and a representative registered client can check for and download patches. Check logs and monitoring for migration failures, authentication errors, and unexpected service errors.

If the update fails, investigate the reported status and logs before proceeding. Follow your recovery plan; do not assume that downgrading the application reverses database migrations.

Update the host operating system, Juju or Kubernetes infrastructure, TLS frontend, database, and storage services through their own supported update procedures. Updating Livepatch Server does not update those independently managed components.

## Monitor releases and get support

Operators are responsible for reviewing the [server release notes](/release-notes/server/index.md) and checking their distribution's `latest/stable` channel regularly. Include these checks in your operational process for identifying and scheduling security updates.

For the release and security-fix policy, see {ref}`Security lifecycle <server-explanation-security-lifecycle>` and [Security support](/server/explanation/architecture/security-overview.md#security-support).

Canonical customers can obtain technical support through the [Canonical support portal](https://portal.support.canonical.com/). See [Get more help](/support/get-more-help.md) for support-ticket guidance.

Report a suspected vulnerability through the {ref}`private vulnerability-reporting procedure <server-how-to-guides-report-a-livepatch-server-vulnerability>`, not a public issue or support discussion. Do not publish credentials or sensitive deployment data in a vulnerability report.
