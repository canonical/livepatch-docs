---
myst:
  html_meta:
    description: "Understand how the Livepatch Client decides whether a reboot into a newer kernel is urgent, and why it sometimes makes no recommendation."
---


(client-explanation-how-kernel-upgrade-recommendations-work)=

# How kernel upgrade recommendations work

When a new kernel is installed, Ubuntu asks for a reboot. This page explains how Livepatch decides whether that reboot is urgent, and why it sometimes makes no recommendation.

## Why the recommendation exists

Live kernel patches cover high and critical kernel vulnerabilities. A newer kernel can include many fixes, but if Livepatch already covers those fixes, the machine isn't exposed while it waits for the next maintenance window. Lower-priority fixes are only available by rebooting into the new kernel.

The kernel upgrade recommendation helps you tell these situations apart. It answers a single question: does the kernel that will boot next fix CVEs that this machine isn't already protected from?

## Where the information comes from

When the Livepatch Server is enabled with the CVE service, it knows which CVEs each Ubuntu kernel version fixes.

On every check-in, the client tells the server:

* The kernel that's running.
* The kernel that will boot after a reboot.
* The CVEs fixed by the live patches that are currently applied.

The server replies with the CVEs that the next boot kernel fixes and that the running kernel doesn't.

## How the client recommends reboots

The client takes the CVEs that the next boot kernel adds and removes the ones that the applied live patches already cover. For example:

* The next boot kernel fixes 5 CVEs that the running kernel doesn't.
* The applied live patches already cover 3 of them.
* The 2 that remain aren't covered, so the client recommends a reboot.

If the applied Livepatch covers all the CVEs fixed by the next boot kernel, the Livepatch Client removes the pending reboot notice, and the reboot can wait for a maintenance window.

## The newest kernel hint

If Canonical has published an even newer kernel in the same series and flavour as the next boot kernel, the client mentions it, together with the CVEs it fixes. This is informational. It never changes the recommendation or whether a reboot notice is removed.

## Why the client sometimes stays quiet

The client only makes a recommendation when it has complete evidence. If anything is uncertain, it makes no recommendation rather than guessing, and it keeps any pending reboot notice. This happens in the following cases:

* The Livepatch Server doesn't provide kernel upgrade information.
* The kernel that will boot next can't be determined with confidence.
* The kernel is a custom or unsigned kernel that doesn't match an installed Ubuntu kernel package.

For the full list, see the [kernel upgrade recommendation reference](/client/reference/platform/kernel-upgrade-recommendation.md).

## Requirements

* Livepatch Client v11.1.0 or later.
* For on-prem deployments, Livepatch Server v2.7.0 and CVE service v1.1.0 or later, with CVE lookup and CVE sync enabled. See [how to deploy the CVE service via snap](/server/how-to-guides/deployment/deploy-cve-service-via-snap.md).

## See also

* [How reboot notices are removed after a kernel update](/client/explanation/architecture/reboot-marker-suppression.md)
* [Kernel upgrade recommendation reference](/client/reference/platform/kernel-upgrade-recommendation.md)
* [Configure the reboot notification policy](/client/how-to-guides/configuration/configure-reboot-notification-policy.md)
