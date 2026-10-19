---
myst:
  html_meta:
    description: "Understand when the Livepatch Client removes the pending reboot notice after a kernel update, and when it keeps it."
---


(client-explanation-how-reboot-notices-are-removed)=

# How reboot notices are removed after a kernel update

With the `livepatch-aware` reboot notification policy, the Livepatch Client can remove the system's pending reboot notice after a kernel install when Livepatch already covers the CVEs fixed by the next boot kernel. This page explains when that happens and when it doesn't.

To configure the policy, see [how to configure the reboot notification policy](/client/how-to-guides/configuration/configure-reboot-notification-policy.md).

## What Livepatch does

After a kernel install, the system shows a pending reboot notice. With the `livepatch-aware` policy, the client checks whether that reboot is needed urgently. After `apt` finishes installing a kernel, a post-invoke hook created by the Livepatch Client refreshes client information and checks the [kernel upgrade recommendation](/client/explanation/architecture/how-kernel-upgrade-recommendations-work.md). It removes the pending reboot notice only when the result proves that Livepatch covers the CVEs fixed by the next boot kernel. The reboot can then wait for a maintenance window, and the Livepatch message shows that the system is protected.

This only applies to the `livepatch-aware` policy.

## When the notice stays

The client keeps the reboot notice in the following cases:

* Packages other than the kernel also need the restart.
* The refresh or the coverage check fails or times out.
* The evidence is incomplete. See [when there is no recommendation](/client/reference/platform/kernel-upgrade-recommendation.md) for the cases.
* The policy is `livepatch-agnostic`.

## Effects on other tools

Removing the reboot notice, i.e. the `/var/run/reboot-required` file, also removes it for other tools that read it. For example:

* `unattended-upgrades` with `Automatic-Reboot` enabled doesn't restart the machine, because it only reboots when a notice is pending.
* `update-notifier` and the message of the day no longer show the system's "System restart required" message.
* Monitoring and fleet management tools that check for the reboot notice no longer report the machine as needing a reboot.

If you don't want Livepatch to change the system's reboot notice, set the policy to `livepatch-agnostic`. See [how to disable Livepatch-aware reboot notifications](/client/how-to-guides/configuration/configure-reboot-notification-policy.md#disable-livepatch-aware-reboot-notifications).

## Limits

* Only kernels installed or upgraded with `apt` are evaluated. A kernel installed with `dpkg -i` isn't evaluated until the next `apt` run.
* If you change the GRUB default or the boot entry later, the notice isn't evaluated again until the next scheduled or manual refresh. The message of the day never shows a recommendation that's out of date.
* On s390x, the client never removes the notice, because the architecture doesn't use GRUB.

## Outcomes at a glance

| Situation after a kernel install | Reboot notice | Livepatch message |
| --- | --- | --- |
| Livepatch covers the next boot kernel's CVEs | Removed | Protected |
| Livepatch doesn't cover all of the next boot kernel's CVEs | Kept | Reboot required |
| Other packages also need the restart | Kept | None, the system's own notice is shown |
| The client can't prove coverage | Kept | None, the system's own notice is shown |

## See also

* [How kernel upgrade recommendations work](/client/explanation/architecture/how-kernel-upgrade-recommendations-work.md)
* [Kernel upgrade recommendation reference](/client/reference/platform/kernel-upgrade-recommendation.md)
* [Configure the reboot notification policy](/client/how-to-guides/configuration/configure-reboot-notification-policy.md)
