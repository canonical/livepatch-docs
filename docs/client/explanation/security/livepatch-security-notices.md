---
myst:
  html_meta:
    description: "Understand Livepatch security notices (LSNs), including when they are issued and how to subscribe to Ubuntu Security Announcements."
---

(client-explanation-livepatch-security-notices)=

# Livepatch security notices

Livepatch Security Notices (LSNs) are notifications issued for kernel vulnerabilities. You can access them through the [Security notices](https://ubuntu.com/security/notices) page or subscribe to the [Ubuntu Security Announcements](https://lists.ubuntu.com/mailman/listinfo/ubuntu-security-announce) mailing list.

LSNs are released for every high or critical kernel vulnerability. They are issued in two scenarios:

- Announcing a new live kernel patch that addresses a vulnerability.
- Alerting users when a live kernel patch cannot be released, describing the reason and possible mitigation. In this case:
  - A standard [Ubuntu Security Notice](https://ubuntu.com/security/notices) (USN) is released with packages to resolve the issue.
  - The Livepatch Client issues a warning that an update and reboot is necessary.
