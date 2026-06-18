---
myst:
  html_meta:
    description: "Understand when a reboot is required for Ubuntu systems using Livepatch, including kernel upgrades and other system components that require restarts."
---

(client-explanation-do-i-need-to-reboot)=

# When to reboot

Live kernel patching is not sufficient when you need to upgrade your kernel to a newer version — a reboot is required in that case. Live kernel patches include only high and critical kernel vulnerabilities. The service provides protection from serious security issues, but this is only one aspect of keeping your system secure and well maintained.

Several other software components can require a system reboot, including:

- CPU firmware and microcode updates
- Updates to shared libraries and low-level dependencies (such as glibc)
- System BIOS and EFI updates

Enabling the Livepatch service does not turn on automatic installation of security updates in APT. For best security, you should:

- [Enable security updates using APT](https://help.ubuntu.com/community/AutomaticSecurityUpdates)
- Subscribe to the [security announcement mailing list](https://lists.ubuntu.com/mailman/listinfo/ubuntu-security-announce)
- Follow all advised security updates and reboot at your earliest convenience when any software component requires it

Kernel SRU updates also include non-security bug fixes and lower-priority security fixes that may be important to your specific circumstances. These fixes are only available by rebooting into an updated kernel.