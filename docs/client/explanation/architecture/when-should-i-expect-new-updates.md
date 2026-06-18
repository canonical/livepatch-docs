---
myst:
  html_meta:
    description: "When to expect new updates - learn about this topic in Livepatch client."
---


(client-explanation-when-should-i-expect-new-updates)=

# When to expect new updates

Live kernel patches are related to and are ''usually'' derived from, but are not the same as, the kernel SRU updates for CVEs. While the vulnerabilities being addressed by both kernel SRU updates and live kernel patches are the same, the process for development and testing are not. A live kernel patch for a vulnerability can be significantly more complex than an ordinary kernel patch, and due to the additional complexity, can take more time to develop and test.

Canonical is committed to live kernel patching every high or critical rated CVE possible as quickly as possible, and in ideal circumstances, a live kernel patch will be available at the same time as the corresponding kernel SRU update.

If a high or critical CVE cannot be remediated with live kernel patching, Livepatch users will be informed at the same time an SRU kernel update that does fix the issue becomes available, in two ways:

- The Livepatch client will indicate that the system must be rebooted by reporting a state of "reboot required." This will be accompanied by a notification in the desktop, on desktop systems, and a notification in the MOTD on the terminal.
- A LSN will be released containing instructions to reboot into a new SRU kernel that contains a fix for the CVE. LSNs are released on both the security website and via email.
