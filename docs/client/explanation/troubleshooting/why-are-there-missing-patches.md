---
myst:
  html_meta:
    description: "Understand why live kernel patches may be missing for certain CVEs, including unsupported kernels, non-kernel vulnerabilities, and third-party driver limitations."
---

(client-explanation-why-was-there-no-livepatch-for-some-particular-high-or-critical-cve)=

# Missing patches

Live kernel patches are produced only for [supported kernels](/client/reference/platform/supported-kernels.md) and only for CVEs that require a Linux kernel modification to resolve the issue. The following scenarios may result in no live kernel patch being available:

- The CVE does not have a kernel fix (for example, a CPU bug may not have a kernel-level fix).
- The security problem exists in Ubuntu software packages rather than the kernel.
- The vulnerability affects third-party drivers that do not ship as part of the Linux kernel (such as NVIDIA GPU drivers).

Livepatch is intended to provide protection in addition to regular kernel security updates. Always read and follow the security advisories published in [Ubuntu Security Notices](https://ubuntu.com/security/notices) (USNs) for your kernel.