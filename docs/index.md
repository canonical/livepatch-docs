---
myst:
  html_meta:
    description: "Livepatch documentation home."
---


# Livepatch Documentation

[Canonical Livepatch](https://ubuntu.com/security/livepatch) patches high and critical Linux kernel vulnerabilities, removing the immediate need to reboot to upgrade the kernel, and instead allowing the downtime to be scheduled. It is a part of the [Ubuntu Pro](https://ubuntu.com/pro) offering.

The Ubuntu Livepatch offering consists of the [client application](/client/index.md), the Livepatch service hosted by Canonical and an optional [on-prem server](/server/index.md). The client runs on machines, periodically checks for available patches, downloads, verifies and installs them.

Canonical Livepatch is meant for critical infrastructure, where unscheduled downtime is to be avoided. By applying live kernel patches for high and critical kernel vulnerabilities, upgrades can be scheduled at a suitable time.

If you're using [Ubuntu Pro](http://ubuntu.com/pro), then you'll have access to two additional Livepatch features.

1. Delayed updates for your [Livepatch clients](/client/index.md),, providing further security and protection.
2. Access to the [on-prem server](/server/index.md).

## [Livepatch Client](/client/index.md)

Livepatch is the client side software that runs on individual machines and periodically checks for the availability of kernel patches. Once a patch becomes available, it is downloaded, verified and applied to the current kernel.

## [Livepatch On-prem](/server/index.md)

Complex enterprise environments often follow policies that require a gradual roll-out of updates to reduce risk, or have high-security isolated environments that need to be updated. Livepatch on-prem allows an organization to define a rollout policy and remain in full control of which machines will get updated and when. To keep your machines up-to-date, the on-premises service regularly syncs with Livepatch hosted by Canonical and obtains the latest patches. It then deploys the patches gradually in as many stages as required.

```{toctree}
:titlesonly:
:maxdepth: 2
:glob:
:hidden:

self
client/index
server/index
```

```{toctree}
:hidden:
:maxdepth: 2

Release Notes <release-notes/index.md>
Contribute <contribute/index.md>
Support <support/index.md>
```