---
myst:
  html_meta:
    description: "Patch Lifecycle - technical reference for Livepatch client."
---


(livepatch-reference-patch-lifecycle-from-cve-to-client-machines)=

# Patch Lifecycle: From CVE to Client Machines

This document outlines the comprehensive lifecycle of kernel live patches, from the initial identification of Common Vulnerabilities and Exposures (CVEs), to their distribution and ongoing monitoring on customer systems. It details the rigorous quality assurance (QA) methods employed to ensure the stability and effectiveness of patches, and how they are delivered and observed in the field.

## CVE Triage and Kernel Patching Process

The process begins with the Canonical Security team, which receives initial CVE reports and provides a list of CVEs affecting kernel packages, and passes them on to the kernel team. The kernel team then triages these CVEs, assigning priority levels (e.g., high, critical). The kernel team also writes the patches for the affected kernels to fix these CVEs. Only patches that fix high or critical severity Linux kernel vulnerabilities are eligible for livepatch. The Livepatch team is responsible for developing kernel livepatches based on the prioritized patches. While the vulnerabilities being addressed by the kernel updates and livepatches are the same, a livepatch for a vulnerability can be significantly more complex than an ordinary kernel patch.

## Livepatch Testing and Release

Livepatches undergo a meticulous testing process. Easy patches are prioritized first, while more complex ones requiring rework or backports are addressed subsequently. Testing involves:

- **Unit tests**: Performed on a virtual machine by loading the livepatch against the same kernel it was written for.
- **Kernel regression tests**: Determine if the livepatch causes any issues while running kernel regression tests. The goal is for the livepatched kernel to pass the same tests as the original, unpatched kernel. These tests are performed on internal machines and monitored for any problems in the patches

Upon successful testing internally, patches are released in phases to the updates tier (e.g., 10%, 20%, 40%, 60%), which consists of free users of Ubuntu Pro. The Livepatch team continuously monitors the metrics for each patch version. This is done using the monitoring support included with the Livepatch client to track patch health and client machine issues. If no crashes are detected and no negative feedback or bug reports are raised, the livepatches are then promoted to the stable tier, which is available to paid Ubuntu Pro users.

## Livepatch Installation

Once the livepatch for a kernel has been released for use, the Livepatch client downloads the patch from the configured Livepatch server and patch source. The downloaded payload consists of the Linux kernel module responsible for patching the running kernel, as well as some metadata. The Livepatch client then inserts the kernel module by making the appropriate syscall. This only affects the in-memory kernel code and not the installed kernel.

The Livepatch client is also equipped with mechanisms to prevent crash loops on installation of a faulty livepatch. For more information on patch installation and preventing crash loops, see [Patch Installation reference](/livepatch/reference/patch-installation.md)

## Livepatch Security

Ensuring the secure transmission and application of a livepatch is paramount to maintaining kernel stability and preventing installation of malicious kernel modules. The Livepatch client ensures that installed livepatches are genuine and authentic before applying them to the kernel. This is achieved by verifying the livepatch content using SHA256 file checksums, verifying the digital signature for the livepatch kernel module using asymmetric encryption and using TLS to communicate with the hosted Canonical Livepatch server.

For more information on how the security of livepatches is ensured, see [Patch Security reference.](/livepatch/reference/patch-security.md)

## Livepatch Monitoring and Blocklisting on faulty patches

In order to [avoid crash loops after application of a faulty livepatch](/livepatch/reference/patch-installation.md), Livepatch client monitors the health of the machine before, during, and after the patching activity. The various phases of Livepatch client's activity include: patch download, patch insertion, patch transition, and patch application.

Livepatch client transmits pings which contain data about patching activity to Livepatch server. These pings are anonymous and are only used to understand the health and status of machines during the insertion process and after the application process.

Insertion pings are sent just before and after the livepatch module has been loaded into the kernel. These pings enable the Livepatch team to monitor the number of livepatch module insertions that were attempted. Note that once loaded, the kernel may not apply the patch immediately if the patch affects functions that are under heavy use.

If the system is running a kernel version above 4.4, the client will move into a transition period where it waits for the kernel to report the patch has been applied. Livepatch client sends transition pings to Livepatch Server which provides data about how many machines have livepatch modules loaded into the kernel, but have not been applied yet. However, if the application of a livepatch results in a kernel crash or a kernel bug within the transition period the error state is reported to the Livepatch server detailing the reason for the failure. Once the client detects the kernel applied the livepatch module, the client proceeds to the the health check pings.

If the system is on kernel version 4.4 or earlier, due to older kernel interfaces, the client will not enter a transition period, and instead check kernel logs for issues after 10 seconds before proceeding to the health check pings.

After a livepatch module is applied, the Livepatch client sends multiple health pings to the hosted Canonical Livepatch server. The first ping will have the time spent waiting for the kernel to apply the livepatch module (if running a kernel later than version 4.4), Then the client sends 3 more health pings from 1 minute, 5 minutes and 15 minutes from the time the kernel applied the livepatch module.

If Livepatch client is retrieving patches from livepatch.canonical.com, the reported metrics are monitored by Canonical's Livepatch team. In the unlikely event that Livepatch client is reporting an error across a significant number of machines, the Livepatch team performs further analysis, and considers halting distribution (blocklisting) of the patch in question. Once a patch has been blocklisted, it will no longer be distributed to new clients or applied in the client machines that already possess the blocklisted patch.
