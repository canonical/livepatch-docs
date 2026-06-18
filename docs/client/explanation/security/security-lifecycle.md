---
myst:
  html_meta:
    description: "Understand the Livepatch Client security lifecycle, including snap release channels, automatic security updates, and managing refresh behavior."
---

(client-explanation-livepatch-client-security-lifecycle)=

# Livepatch Client security lifecycle

The Livepatch Client is released as a snap. Releases are issued on an ad-hoc basis as features and bug fixes are implemented. When a security vulnerability is detected in an upstream dependency, a best-effort attempt is made to upgrade the dependency to the latest version that resolves the vulnerability. This security fix is then included in the next ad-hoc release.

It is recommended to install the [canonical-livepatch snap](https://snapcraft.io/canonical-livepatch) from the `latest/stable` channel in the Snap Store. This channel provides the latest stable version of the snap, including all recent stable security updates.

## Security updates

Security updates are released to the `latest/stable` channel of the canonical-livepatch snap. By default, these updates are automatically applied by the snapd daemon, which checks for updates four times a day and applies them when available. Each update check is called a **refresh**.

You can control the default update behavior of snapd using the `snap refresh` command. This includes:

- Manually checking for and applying updates
- Postponing updates for a period of time or indefinitely
- Scheduling updates
- Rolling back automatically applied updates

For more information, see the [snap documentation on managing updates](https://snapcraft.io/docs/how-to-guides/manage-snaps/manage-updates/#refresh-update-control).

Modifying the default auto-update behavior of snapd could leave systems vulnerable to important security updates. To verify that a security update has been applied, run `snap info canonical-livepatch` and confirm that the installed snap version matches the version in the `latest/stable` channel. If the versions do not match, manually apply the most recent updates:

```
sudo snap refresh canonical-livepatch --channel=latest/stable
```
