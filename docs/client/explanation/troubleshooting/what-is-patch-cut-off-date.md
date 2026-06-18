---
myst:
  html_meta:
    description: "Understand the patch cut-off date feature in Livepatch, including how it controls patch application, availability requirements, and security risk warnings."
---

(client-explanation-what-is-patch-cut-off-date)=

# Patch cut-off date

The patch cut-off date is a feature that allows you to set a point in time after which no patches will be applied to the system. This is useful for ensuring that the system state is deterministic and reproducible, guaranteeing that no changes occur after a specific date.

The use of a patch cut-off date is recommended only for groups of systems that require a high level of uniformity and synchronized updates. Delaying the application of high and critical security patches leaves the exploit window of a known vulnerability open until the patch is applied.

## Availability

This feature is available only for users with a paid Ubuntu Pro subscription, or to public cloud customers running Ubuntu Pro images that retrieve Livepatch patches from Canonical's hosted Livepatch service. It is not available for self-hosted Livepatch Servers.

Livepatch Client version 10.11.2 or later is required.

## Excluded CVE fixes

Starting from Livepatch Client version 10.15.0, verbose output includes a warning message if the cut-off date has blocked the latest patch. This message lists the CVEs the machine is no longer protected against, along with the related LSN and LSN publish timestamp.

Example output from running `canonical-livepatch status --verbose` with an older patch:

```
[!] KERNEL PATCHES BLOCKED: SECURITY RISK DETECTED
The latest patches for your kernel have been blocked by current configuration.
An older patch has been installed instead.
Run "canonical-livepatch config" and review the values for cutoff-date to check if they are still relevant.
BLOCKED SECURITY UPDATES:
CVE ID                 Published                      Related LSN
UBUNTU-CVE-2024-26800  2024-12-19 11:12:01 +0000 UTC  LSN-0108-1
UBUNTU-CVE-2024-26921  2024-12-19 11:12:01 +0000 UTC  LSN-0108-1
UBUNTU-CVE-2024-26960  2024-12-19 11:12:01 +0000 UTC  LSN-0108-1
UBUNTU-CVE-2024-27398  2024-12-19 11:12:01 +0000 UTC  LSN-0108-1
UBUNTU-CVE-2024-38630  2024-12-19 11:12:01 +0000 UTC  LSN-0108-1
UBUNTU-CVE-2024-43882  2024-12-19 11:12:01 +0000 UTC  LSN-0108-1
UBUNTU-CVE-2024-50264  2024-12-19 11:12:01 +0000 UTC  LSN-0108-1
UBUNTU-CVE-2024-26800  2025-02-20 10:11:03 +0000 UTC  LSN-0109-1
UBUNTU-CVE-2024-26921  2025-02-20 10:11:03 +0000 UTC  LSN-0109-1
UBUNTU-CVE-2024-38630  2025-02-20 10:11:03 +0000 UTC  LSN-0109-1
UBUNTU-CVE-2024-43882  2025-02-20 10:11:03 +0000 UTC  LSN-0109-1
UBUNTU-CVE-2024-50264  2025-02-20 10:11:03 +0000 UTC  LSN-0109-1
UBUNTU-CVE-2024-53103  2025-02-20 10:11:03 +0000 UTC  LSN-0109-1
UBUNTU-CVE-2023-52880  2025-03-26 09:20:22 +0000 UTC  LSN-0110-1
UBUNTU-CVE-2024-38558  2025-03-26 09:20:22 +0000 UTC  LSN-0110-1
UBUNTU-CVE-2024-53104  2025-03-26 09:20:22 +0000 UTC  LSN-0110-1
UBUNTU-CVE-2024-53140  2025-03-26 09:20:22 +0000 UTC  LSN-0110-1
UBUNTU-CVE-2024-56672  2025-03-26 09:20:22 +0000 UTC  LSN-0110-1
UBUNTU-CVE-2025-0927   2025-03-26 09:20:22 +0000 UTC  LSN-0110-1
```

## What happens if a patch has already been applied

If you already have a patch applied and its release date is after the cut-off date, you must reboot the machine to fully remove the changes.

If the patch release date is before the cut-off date, no action is required — the patch will remain applied.