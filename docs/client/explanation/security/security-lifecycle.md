---
myst:
  html_meta:
    description: "Security Lifecycle - learn about this topic in Livepatch client."
---


(client-explanation-livepatch-client-security-lifecycle)=

# Livepatch Client Security Lifecycle

The Livepatch Client is released as a snap. The releases for the client are done ad-hoc as a number of features and bug fixes are implemented. When a security vulnerability in an upstream dependency is detected, a best-case effort is made to upgrade the dependency to the latest version that fixes the vulnerability. This security fix is then included in the next ad-hoc release.

It is recommended that the [canonical-livepatch snap](https://snapcraft.io/canonical-livepatch) is installed from the `latest/stable` channel from the snapstore. This channel will hold the latest stable version for the snap, including receiving all of the most recent stable security updates.

## Security Updates

Security updates will be released to the `latest/stable` channel of the canonical-livepatch snap. By default, these security updates will be automatically applied to the installed canonical-livepatch snap by the snapd daemon. The snapd daemon checks for these updates four times a day and applies an update when it is available. Each of these update checks is called a **refresh**.

The default update behaviour of the snapd daemon can be controlled using the `snap refresh` command. This includes manually checking for and applying updates, postponing updates for a period of time or indefinitely, scheduling updates and rolling back automatically applied updates. For more information, view the [snap documentation](https://snapcraft.io/docs/how-to-guides/manage-snaps/manage-updates/#refresh-update-control) on managing snap updates.

Please note that, modifying the default auto-update behavior of the snapd daemon, could leave systems vulnerable to important security updates shipped to the snaps. To verify that a security update has been successfully applied, run `snap info canonical-livepatch` and ensure that the installed snap version matches the snap version in the `latest/stable` channel. If the versions do not match, manually apply the most recent updates by running the following command:

```
sudo snap refresh canonical-livepatch --channel=latest/stable
```
