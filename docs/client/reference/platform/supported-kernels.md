---
myst:
  html_meta:
    description: "Reference of kernel versions, flavours, and Ubuntu releases with Livepatch support, including upgrade and reboot interval guidance."
---

(client-reference-kernels-covered-by-livepatch)=

# Kernels covered by Livepatch

The following table lists the kernel versions, flavours, and Ubuntu releases for which live kernel patches are available.

| Ubuntu release | Architecture | Kernel version | Kernel flavours | Upgrade and reboot interval* |
| -------------- | ------------ | -------------- | --------------- | --------------------------- |
| Ubuntu 26.04 LTS | arm64 | 7.0 (GA) | aws, azure, fips, gcp, generic, gke, ibm, lowlatency, oracle | every 13 months |
| Ubuntu 26.04 LTS | 64-bit x86 | 7.0 (GA) | aws, azure, fips, gcp, generic, gke, ibm, ibm-gt, ibm-gt-tdx, lowlatency, oracle | every 13 months |
| Ubuntu 24.04 LTS | 64-bit x86 | 7.0 (HWE) | aws, azure, fips, gcp, generic, gke, ibm | every 13 months |
| Ubuntu 24.04 LTS | 64-bit x86 | 6.17 (HWE) | aws, azure, gcp, generic, gke, ibm, oracle | every 9 months |
| Ubuntu 24.04 LTS | 64-bit x86 | 6.14 (HWE) | generic | every 9 months |
| Ubuntu 24.04 LTS | 64-bit x86 | 6.11 (HWE) | aws, azure, gcp, generic, gke, ibm, oracle | every 9 months |
| Ubuntu 24.04 LTS | 64-bit x86 | 6.8 (GA) | aws, azure, fips, gcp, generic, gke, ibm, lowlatency | every 13 months |
| Ubuntu 24.04 LTS | s390x | 6.8 (GA) | generic | every 13 months |
| Ubuntu 22.04 LTS | 64-bit x86 | 6.8 (HWE) | aws, azure, gcp, generic, gke, ibm, lowlatency | every 13 months |
| Ubuntu 22.04 LTS | 64-bit x86 | 5.15 (GA) | aws, aws-fips, azure, azure-fips, fips, gcp, gcp-fips, generic, gke, ibm, lowlatency, oracle | every 13 months |
| Ubuntu 22.04 LTS | s390x | 5.15 (GA) | generic | every 13 months |
| Ubuntu 20.04 LTS | 64-bit x86 | 5.15 (HWE) | aws, azure, gcp, generic, gke, ibm, lowlatency, oracle | every 13 months |
| Ubuntu 20.04 LTS | 64-bit x86 | 5.4 (GA) | aws, aws-fips, azure, azure-fips, fips, gcp, gcp-fips, generic, gke, gkeop, ibm, lowlatency, oem, oracle | every 13 months |
| Ubuntu 18.04 LTS | 64-bit x86 | 5.4 (HWE) | aws, azure, gcp, generic, gke, gkeop, ibm, lowlatency, oracle | every 13 months |
| Ubuntu 18.04 LTS | 64-bit x86 | 4.15 (GA) | aws, aws-fips, azure, azure-fips, fips, gcp, gcp-fips, generic, gke, lowlatency, oem, oracle | every 13 months |
| Ubuntu 16.04 LTS | 64-bit x86 | 4.15 (HWE) | azure, generic, lowlatency | every 13 months |
| Ubuntu 16.04 LTS | 64-bit x86 | 4.4 (GA) | aws, fips, generic, lowlatency | every 13 months |
| Ubuntu 14.04 LTS | 64-bit x86 | 4.4 (HWE) | generic, lowlatency | every 13 months |

**\*Upgrade and reboot interval:** Security patches are created for a kernel for up to 9–13 months from its release date. Upgrade and restart the machine within this period to continue receiving Livepatch patches. New kernel ABIs are provided during the [Ubuntu Pro security coverage period](https://ubuntu.com/pro).

GA refers to the kernel a release launched with. [HWE (Hardware Enablement)](https://ubuntu.com/kernel/lifecycle) kernels are newer kernels introduced into the current LTS release as they ship with subsequent Ubuntu versions, up until the next LTS release.

Livepatch provides HWE kernel support for a limited combination of kernel flavour, kernel version, and Ubuntu release, as described in the table above.

Livepatch provides security coverage for ARM64 (arm64) architectures from release 26.04 onward. Older releases are not covered for the ARM64 architecture due to limitations in the available tooling.

For help diagnosing why a specific kernel may not be receiving patches, see the [troubleshooting guide](/client/explanation/troubleshooting/why-livepatch-is-not-working-on-my-machine.md).