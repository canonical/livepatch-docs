---
myst:
  html_meta:
    description: "Understand the full lifecycle of live kernel patches in Livepatch, from CVE triage and development through testing, release, installation, and ongoing monitoring."
---

(client-explanation-patches-patch-lifecycle-from-cve-to-client-machines)=

# Patch lifecycle: from CVE to client machines

This document outlines the complete lifecycle of kernel live patches, from the initial identification of Common Vulnerabilities and Exposures (CVEs) through to distribution and ongoing monitoring on customer systems. It covers the quality assurance (QA) methods used to ensure patch stability and effectiveness, and how patches are delivered and observed in the field.

## CVE triage and kernel patching

The process begins with the Canonical Security team, which receives initial CVE reports, identifies CVEs affecting kernel packages, and forwards them to the kernel team. The kernel team triages these CVEs, assigning priority levels such as high or critical, and writes patches for the affected kernels. Only patches that fix high or critical severity Linux kernel vulnerabilities are eligible for live kernel patching.

The Livepatch team develops live kernel patches based on the prioritized kernel patches. While the vulnerabilities addressed by kernel updates and live kernel patches are the same, a live kernel patch for a vulnerability can be significantly more complex than an ordinary kernel patch.

## Testing and release

Live kernel patches undergo a thorough testing process. Simpler patches are prioritized first, while more complex ones requiring rework or backports are addressed subsequently. Testing includes:

- **Unit tests**: Performed on a virtual machine by loading the live kernel patch against the same kernel it was written for.
- **Kernel regression tests**: Verify that the live kernel patch does not introduce issues when running kernel regression tests. The objective is for the live-patched kernel to pass the same tests as the original, unpatched kernel. These tests run on internal machines and are monitored for any problems.

After successful internal testing, patches are released in phases to the Updates tier (for example, 10%, 20%, 40%, 60%), which serves free users of Ubuntu Pro. The Livepatch team continuously monitors metrics for each patch version using the monitoring support included with the Livepatch Client to track patch health and client machine issues. If no crashes are detected and no negative feedback or bug reports are raised, the patches are promoted to the Stable tier, available to paid Ubuntu Pro users.

## Installation

Once a live kernel patch has been released, the Livepatch Client downloads it from the configured Livepatch Server and patch source. The downloaded payload consists of the Linux kernel module responsible for patching the running kernel, along with metadata. The Livepatch Client inserts the kernel module by making the appropriate syscall. This affects only the in-memory kernel code, not the installed kernel.

The Livepatch Client includes mechanisms to prevent crash loops when installing a faulty live kernel patch. For more information on patch installation and crash loop prevention, see the [Patch installation reference](/client/explanation/patches/patch-installation.md).

## Security

Ensuring secure transmission and application of live kernel patches is essential to maintaining kernel stability and preventing the installation of malicious kernel modules. The Livepatch Client verifies that installed patches are genuine before applying them to the kernel:

- Patch content is verified using SHA256 file checksums.
- The digital signature for the Livepatch kernel module is verified using asymmetric encryption.
- Communication with the hosted Canonical Livepatch Server uses TLS.

For more information on how patch security is ensured, see the [Patch security reference](/client/reference/patches/patch-security.md).

## Monitoring and blocklisting

To prevent crash loops after applying a faulty live kernel patch, the Livepatch Client monitors machine health before, during, and after patching activity. The phases of client activity include patch download, patch insertion, patch transition, and patch application.

The Livepatch Client transmits pings containing data about patching activity to the Livepatch Server. These pings are anonymous and are used solely to understand the health and status of machines during and after the patching process.

**Insertion pings** are sent immediately before and after the Livepatch module is loaded into the kernel. These pings allow the Livepatch team to monitor the number of insertion attempts. After loading, the kernel may not apply the patch immediately if the affected functions are under heavy use.

If the system is running a kernel version above 4.4, the client enters a **transition period** where it waits for the kernel to report that the patch has been applied. The client sends transition pings to the Livepatch Server, providing data about machines with loaded but unapplied patch modules. If patch application results in a kernel crash or bug during the transition period, the error state is reported to the Livepatch Server with details about the failure. Once the client detects the kernel has applied the module, it proceeds to health check pings.

If the system is on kernel version 4.4 or earlier, the client does not enter a transition period due to older kernel interfaces. Instead, it checks kernel logs for issues after 10 seconds before proceeding to health check pings.

After a Livepatch module is applied, the client sends multiple **health pings** to the hosted Canonical Livepatch Server. The first ping includes the time spent waiting for the kernel to apply the module (for kernels later than version 4.4). The client then sends three additional health pings at 1 minute, 5 minutes, and 15 minutes from the time the kernel applied the module.

If the client retrieves patches from `livepatch.canonical.com`, the reported metrics are monitored by Canonical's Livepatch team. If the client reports errors across a significant number of machines, the Livepatch team performs further analysis and considers halting distribution (blocklisting) of the patch. Once a patch is blocklisted, it is no longer distributed to new clients or applied on client machines that already possess it.
