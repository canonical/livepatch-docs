---
myst:
  html_meta:
    description: "Check for and apply Livepatch Server security updates, manage standalone snap automatic updates, and obtain support or report vulnerabilities."
---

(server-how-to-guides-get-security-updates-and-support)=

# Get security updates and support

This guide covers updates to the Livepatch Server application, not the kernel patches that it distributes to clients. Offline patch imports do not update the server; deployments without store access need to deliver server updates through their offline deployment process. The Juju examples assume that the application is named `livepatch`.

Before updating, review the release notes, plan for service restarts, and back up the database, deployment configuration, and any patch data that your recovery plan requires. Updates can apply database migrations automatically; do not assume that reverting the charm or snap reverses them. After updating, check that the expected revision is installed, the application is healthy, an administrator can authenticate, and a test client can download patches. If the update fails, check the status and logs and follow your recovery plan.

## Update a Juju-managed deployment

This procedure applies to the current machine charm and the Kubernetes charm. For a legacy reactive machine charm or the original Operator machine charm on `ops1.x/*`, first follow {ref}`Migrate to the new machine charm <server-how-to-guides-migrate-to-the-new-machine-charm>` instead of refreshing directly to `latest/stable`.

Check the deployed revision with `juju status livepatch` and compare it with `latest/stable` on Charmhub for the [machine charm](https://charmhub.io/canonical-livepatch-server) or [Kubernetes charm](https://charmhub.io/canonical-livepatch-server-k8s). Then apply the current stable revision:

```shell
juju refresh livepatch --channel=latest/stable
```

Wait for the charm to finish updating and check its status again. Follow any release-specific instructions, but do not routinely run a manual `schema-upgrade` action after a successful update. The machine charm manages its server snap and holds its automatic refreshes, so do not refresh or unhold that snap manually. Likewise, update the Kubernetes charm rather than replacing its workload image. Juju does not refresh charms automatically by default, so schedule regular checks. If you automate charm refreshes, control postponement in that automation rather than with snapd.

## Update a standalone snap manually

This procedure does not apply to the snap managed by the machine charm. Check the installed revision, tracked channel, and available stable revision, then apply the current stable revision:

```shell
snap list canonical-livepatch-server
snap info canonical-livepatch-server
sudo snap refresh canonical-livepatch-server --channel=latest/stable
```

## Enable automatic updates

By default, snapd checks for updates four times a day and applies them automatically. To enable automatic updates for a standalone snap, make sure that it tracks `latest/stable` and remove any hold with `sudo snap refresh --unhold canonical-livepatch-server`.

## Postpone or schedule updates

To postpone automatic updates for 24 hours, run `sudo snap refresh --hold=24h canonical-livepatch-server`. To allow refreshes only between 02:00 and 04:00, run `sudo snap set system refresh.timer=02:00-04:00`. This setting affects all snaps on the host. Check the resulting schedule with `snap refresh --time`.

## Disable automatic updates

To hold automatic updates indefinitely, run `sudo snap refresh --hold=forever canonical-livepatch-server`. Known vulnerabilities then stay unpatched until you update manually, so use this only with an operator-managed process for checking releases and applying fixes. An explicit refresh can still update a held snap. See the [snap update-control documentation](https://snapcraft.io/docs/how-to-guides/manage-snaps/manage-updates/#refresh-update-control) for more on holds and schedules.

## Monitor releases and get support

To learn about new security updates, review the [server release notes](/release-notes/server/index.md) and check the `latest/stable` channel regularly. For the release and security-fix policy, see {ref}`Security lifecycle <server-explanation-security-lifecycle>` and [Security support](/server/explanation/architecture/security-overview.md#security-support).

Canonical customers can get technical support through the [Canonical support portal](https://portal.support.canonical.com/). See [Get more help](/support/get-more-help.md) for support-ticket guidance. Report suspected vulnerabilities privately through {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`, not in a public issue.
