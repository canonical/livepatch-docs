---
myst:
  html_meta:
    description: "Supported kernels - technical reference for Livepatch client."
---


(client-reference-kernels-covered-by-livepatch)=

# Kernels covered by Livepatch

| Ubuntu release | Arch | Kernel Version | Kernel Variants | Upgrade and Reboot\* |
| ---------------- | ---------- | -------------- | -------------------------------------------------------------------------------------------------------- | ------------------- |
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

**Upgrade and Reboot Interval:** Security patches are only created for a kernel for up to 9-13 months from the release date of the kernel. We recommend updating and restarting your machine within this period to continue receiving Livepatch updates. New kernel ABIs are provided during the Ubuntu Pro security coverage period, which you can learn more about [here](https://ubuntu.com/pro).

GA is the kernel a release launched with, while [HWE or Hardware Enablement](https://ubuntu.com/kernel/lifecycle) kernels are a set of newer kernel that become available in the current LTS release as these newer kernels are released with subsequent Ubuntu versions, up until the next LTS release.

There will be Livepatch support for HWE kernels across a limited combination of kernel flavour (variants), kernel version, and Ubuntu release as detailed above.

Livepatch will provide security coverage for ARM64 architectures from release 26.04. Older releases are not covered for the ARM64 architecture, due to limitations in the tooling available in these releases.

See [this page](/client/explanation/troubleshooting/why-livepatch-is-not-working-on-my-machine.md) to better understand why your kernel might not be supported.
