---
myst:
  html_meta:
    description: "Report bugs for the Livepatch Client and server, including diagnostic commands and privacy recommendations."
---

(support-reporting-bugs)=

# Report bugs

This page describes how to report bugs for the Livepatch Client and the Livepatch on-premises server. Including the right diagnostic information in your bug report helps the development team diagnose and resolve your issue quickly.

## Report a Livepatch Client bug

File client bugs on the [canonical-livepatch Launchpad project](https://bugs.launchpad.net/canonical-livepatch-client/+filebug). Include the output from the following commands in your bug report:

```bash
snap info canonical-livepatch
```

```bash
canonical-livepatch status
```

```bash
lsb_release -a
```

```bash
uname -a
```

```bash
journalctl -u snap.canonical-livepatch.canonical-livepatchd
```

The output of `journalctl` can be lengthy. To include only recent log entries, pipe the output into `tail`:

```bash
journalctl -u snap.canonical-livepatch.canonical-livepatchd | tail -100
```

**Note:** Mark the bug as private if any of the diagnostic output contains personal or sensitive information that you do not want publicly visible.

## Report a Livepatch on-prem server bug

File server bugs on the [livepatch-onprem Launchpad project](https://bugs.launchpad.net/livepatch-onprem/+filebug). Include the server version and relevant log output in your bug report.
