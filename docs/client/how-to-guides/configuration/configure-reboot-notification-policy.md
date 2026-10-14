---
myst:
  html_meta:
    description: "How to configure the reboot notification policy of the Livepatch Client, so that reboot notices account for kernel CVEs already covered by Livepatch."
---


(client-how-to-guides-how-to-configure-the-reboot-notification-policy)=

# How to configure the reboot notification policy

By default, the Livepatch Client can remove a pending reboot notice after a kernel update when Livepatch already covers the CVEs that the next boot kernel fixes. This guide shows how to check, enable, and disable that behavior.

## Requirements

* Livepatch Client v11.1.0 or later.
* For on-prem deployments, Livepatch Server v2.7.0 and CVE service v1.1.0 or later.

## Check the reboot notification policy

To check the reboot notification policy, run the following command:

```bash
sudo canonical-livepatch config reboot-notification-policy --format yaml
reboot-notification-policy: livepatch-aware

sudo canonical-livepatch config reboot-notification-policy --format json
{"reboot-notification-policy":"livepatch-aware"}
```

## Enable Livepatch-aware reboot notifications

To make reboot notifications aware of the Livepatch state, which is the default, run the following command:

```bash
sudo canonical-livepatch config reboot-notification-policy=livepatch-aware
```

With this policy, the client removes a pending reboot notice only after a refresh confirms that Livepatch covers the CVEs fixed by the next-boot kernel. If the refresh, the coverage check, or either command's timeout fails, the notice stays.

The snap connects the interfaces that this feature needs (`etc-kernel-postinst-d`, `etc-apt-conf-d`, and `boot-grub`) automatically. If the client does not remove notices as expected, check that all three interfaces are connected:

```bash
snap connections canonical-livepatch
```

To connect a missing interface, run:

```bash
sudo snap connect canonical-livepatch:<interface>
```

## Disable Livepatch-aware reboot notifications

When the client removes a reboot notice, tools that read it, such as `unattended-upgrades` and `update-notifier`, are affected. If you don't want the client to remove reboot notices, make reboot notifications independent of the Livepatch state by running the following command:

```bash
sudo canonical-livepatch config reboot-notification-policy=livepatch-agnostic
```

With this policy, the client never removes a pending reboot notice.

## Next steps

* To learn how the client decides to remove a reboot notice, see [how reboot notices are removed after a kernel update](/client/explanation/architecture/reboot-marker-suppression.md).
* For all client settings, see the [configuration options](/client/reference/platform/configuration-options.md).
