---
myst:
  html_meta:
    description: "Livepatch security notices - learn about this topic in Livepatch client."
---


(client-explanation-livepatch-security-notices)=

# Livepatch security notices

The Livepatch Security Notices (LSN) are notifications issued for kernel vulnerabilities. They can be accessed by following our [Security notices](https://ubuntu.com/security/notices) or by subscribing to the [Ubuntu Security Announcements](https://lists.ubuntu.com/mailman/listinfo/ubuntu-security-announce) mailing list. LSNs are released for every high or critical kernel vulnerability. They are released for:

- Announcing a new livepatch addressing a vulnerability.
- Communicating an alert if a livepatch cannot be released describing the reason and possible mitigation. In that case
  - a standard [Ubuntu security notice](https://ubuntu.com/security/notices) (USN) will be released with packages along side it to fix the issue.
  - the livepatch client will issue a warning that an update and reboot is necessary.
