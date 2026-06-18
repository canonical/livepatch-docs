---
myst:
  html_meta:
    description: "Understand when to expect new Livepatch updates, including the relationship between kernel SRU updates and live kernel patches, and notification mechanisms."
---

(client-explanation-when-should-i-expect-new-updates)=

# When to expect new updates

Live kernel patches are related to kernel SRU updates for CVEs and are usually derived from them, but are not the same. While the vulnerabilities addressed by both kernel SRU updates and live kernel patches are identical, the development and testing processes differ. A live kernel patch for a vulnerability can be significantly more complex than an ordinary kernel patch and may require additional time to develop and test.

Canonical is committed to live patching every high or critical CVE as quickly as possible. In ideal circumstances, a live kernel patch is available at the same time as the corresponding kernel SRU update.

If a high or critical CVE cannot be fixed through live kernel patching, Livepatch users are informed when the SRU kernel update that resolves the issue becomes available. Two notifications are issued:

- The Livepatch client indicates that the system must be rebooted by reporting a state of "reboot required." On desktop systems, this is accompanied by a desktop notification. On all systems, a notification appears in the MOTD on the terminal.
- An LSN is released containing instructions to reboot into a new SRU kernel that includes the CVE fix. LSNs are published on both the security website and via email.
