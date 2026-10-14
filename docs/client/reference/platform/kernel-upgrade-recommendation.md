---
myst:
  html_meta:
    description: "Reference for the Livepatch kernel upgrade recommendation, including its results, messages, message of the day rules, and status output fields."
---


(client-reference-kernel-upgrade-recommendation)=

# Kernel upgrade recommendation

The Livepatch Client produces a kernel upgrade recommendation. It appears in the message of the day (MOTD) and helps the client decide whether to remove a pending reboot notice. This page describes the possible results, the messages, and the related `status` output.

For the reasoning behind the recommendation, see [how kernel upgrade recommendations work](/client/explanation/architecture/how-kernel-upgrade-recommendations-work.md).

## Requirements

* Livepatch Client v11.1.0 or later.
* For on-prem deployments, Livepatch Server v2.7.0 and CVE service v1.1.0 or later.

## Recommendation results

The recommendation is the exit status of `canonical-livepatch kernel-upgrade-required`, the command that the Livepatch MOTD script and reboot notice handling use internally.

| Exit status | Meaning |
| --- | --- |
| `0` | A reboot is recommended because of an unpatchable or uncovered vulnerability, an expired Livepatch SRU window, or the end of kernel series support. |
| `1` | Livepatch covers the CVEs that matter for the running kernel. The reboot can wait for a maintenance window. |
| `2` | The client doesn't have enough information to make a recommendation. |
| `3` | An error prevented the client from determining a result. |

### When there is no recommendation

The client returns exit status `2` instead of guessing in the following cases:

* The Livepatch Server is older than v2.7.0 and doesn't send kernel upgrade information.
* The next boot kernel can't be determined with confidence. For example, a one-time boot entry or a custom or saved GRUB default is set, or the GRUB configuration can't be read.
* The kernel is a custom kernel, or an unsigned kernel, that doesn't match an installed `linux-image-*` package.
* The machine doesn't use GRUB, as on s390x.
* No recommendation applies, for example when the running kernel is already the newest installed kernel.

## Messages

### Reboot recommended

When a reboot is recommended and Livepatch is applied, the message names the next boot kernel and counts the CVEs it fixes that neither the running kernel nor the applied live patches fix. The counts are grouped by severity:

```text
*** Kernel upgrade and reboot required.

Canonical Livepatch has applied live patches for 3 CVEs in the running kernel.

Rebooting will load kernel 6.8.0-50-generic, which has patches for
5 CVEs not fixed in the running kernel or by the applied livepatch:
  1 critical + 4 high = 5 CVEs

Consider rebooting. ***
```

When Canonical has published an even newer kernel in the same series and flavour as the next boot kernel, the message adds a paragraph about the CVEs fixed in that latest kernel:

```text
Canonical has published kernel 6.8.0-52-generic, which has patches for
19 CVEs not fixed in the running kernel or by the applied livepatch:
  2 critical + 5 high + 12 medium = 19 CVEs

Consider upgrading to the latest available kernel, and rebooting. ***
```

The information about the newest kernel is only informational. It never changes the recommendation result or the reboot notice.

### Protected

When Livepatch covers the running kernel, the message lists up to three of the CVEs fixed by the applied live patches:

```text
*** Canonical Livepatch has protected this system against CVE-2024-1, CVE-2024-2 and CVE-2024-3 (and 2 more). Upgrade to the latest secure kernel before 2025-06-01 ***
```

The date is the end of the kernel's SRU security coverage. If the date isn't available, the message ends with `Upgrade to the latest secure kernel at a suitable time` instead.

## Message of the day

The MOTD shows the recommendation as follows:

| Result | When the message is shown |
| --- | --- |
| Exit status `0` | Always, whichever reboot notification policy is set. |
| Exit status `1` or `2` | When the result includes a message, no system reboot notice is pending, and the [reboot notification policy](/client/how-to-guides/configuration/configure-reboot-notification-policy.md) is `livepatch-aware`. |
| Exit status `3`, or a timeout | Never. |

When a system reboot notice is pending, the system's own "System restart required" message is shown instead of the Livepatch protected message, so that the two messages don't contradict each other.

## Status output

The `kernel-upgrade` block in the `status` output shows the evidence behind the recommendation. It's available with `--format json` and `--format yaml`:

```yaml
kernel-upgrade:
  status: upgrade-available
  running-kernel: linux-image-6.8.0-40-generic
  next-boot-kernel: linux-image-6.8.0-50-generic
  fixed-by-next-boot-kernel-version-summary:
    critical: 1
    high: 4
    medium: 30
  latest-kernel: linux-image-6.8.0-52-generic
  fixed-by-latest-kernel-version-summary:
    critical: 2
    high: 5
    medium: 40
```

| Field | Description |
| --- | --- |
| `status` | `upgrade-available` when the next boot kernel fixes CVEs that the running kernel doesn't, `not-available` when it fixes none, or `same-kernel` when the running and next boot kernels are the same. |
| `running-kernel` | The kernel package that's currently running. |
| `next-boot-kernel` | The kernel package that will boot after a reboot. |
| `fixed-by-next-boot-kernel-version-summary` | The number of CVEs fixed by the next boot kernel and not by the running kernel, per severity. |
| `latest-kernel` | A newer kernel in the same series and flavour as the next boot kernel. Shown only when it exists. |
| `fixed-by-latest-kernel-version-summary` | The number of CVEs fixed by the latest kernel and not by the running kernel, per severity. Shown only with `latest-kernel`. |

Severities with no CVEs are omitted. With `--verbose`, the full `fixed-by-next-boot-kernel-version` and `fixed-by-latest-kernel-version` CVE lists are shown too. The default `yaml` or `json` output doesn't show these fields.

## See also

* [How reboot notices are removed after a kernel update](/client/explanation/architecture/reboot-marker-suppression.md)
* [Check the Livepatch Client status](/client/how-to-guides/operations/check-client-status.md)
