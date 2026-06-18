---
myst:
  html_meta:
    description: "Filing support tickets and getting help with the Livepatch client and on-premises server."
---

(support-how-do-i-get-more-help)=

# Get more help

Access to the Ubuntu Livepatch service requires an [Ubuntu Pro](https://ubuntu.com/pro) subscription. Ubuntu Pro includes Canonical support for the Livepatch client, the Livepatch on-premises server, and the hosted Livepatch service.

## File a support ticket

Ubuntu Pro customers can file support tickets through the [Canonical support portal](https://portal.support.canonical.com/).

When filing a support ticket, include the following information to help the support team diagnose your issue:

- Livepatch client version (`canonical-livepatch --version`)
- Livepatch client machine token (`canonical-livepatch status --verbose`)
- Ubuntu release and kernel version (`lsb_release -a` and `uname -a`)
- Relevant log output from the Livepatch client daemon (`journalctl -u snap.canonical-livepatch.canonical-livepatchd`)

If you are reporting an issue with the Livepatch on-premises server, include the server version and relevant server log output in your ticket.
