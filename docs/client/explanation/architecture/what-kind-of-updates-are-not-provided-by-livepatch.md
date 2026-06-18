---
myst:
  html_meta:
    description: "Understand which updates are not provided by the Livepatch service, including bug fixes, performance improvements, and driver updates outside of high and critical CVEs."
---

(client-explanation-what-kind-of-updates-are-not-provided-by-the-livepatch-service)=

# Updates not provided by Livepatch

The Livepatch service provides patches exclusively for Canonical-released kernels, addressing security issues that have been assigned a CVE and are rated as high or critical priority.

Live kernel patches are intended to address significant security issues in the kernel and provide protection from serious vulnerabilities until a reboot can be scheduled. All CVEs rated as high or critical will be fixed through live kernel patching whenever possible. If a live patch cannot be produced, the Livepatch Client issues a notification that a reboot is required.

Many patches included in [kernel updates](https://wiki.ubuntu.com/KernelTeam/KernelUpdates) are outside the scope of the Livepatch service, including:

- Bug fixes that are not security issues
- Performance improvements
- Driver updates
- New features

To receive these updates, upgrade the kernel package using your system's package manager — `apt` for Ubuntu Desktop or Server, or `snapd` for Ubuntu Core — then reboot into the upgraded kernel.
