---
myst:
  html_meta:
    description: "How to check client status with Livepatch client."
---

(client-how-to-guides-how-to-check-the-livepatch-client-status)=

# How to check the Livepatch client status

Once `canonical-livepatch`, the Livepatch client, is running on a machine, it will periodically (every hour by default) check for new patches.

To show the current state of the client, run:

```
canonical-livepatch status
```

Example output:

```
last check: 52 seconds ago
kernel: 5.4.0-216.236-generic
server check-in: succeeded
kernel state: ✓ kernel series 5.4 is covered by Livepatch
patch state: ✓ all applicable live kernel patch modules applied
patch version: 113.1
tier: updates (Free usage; This machine beta tests new patches.)
machine id: {alpha-numeric-string}
```

The `kernel state` line indicates the current security coverage status for your running kernel:

- **✓ kernel series {kernel-series} is covered by Livepatch**

  The kernel series (e.g. `5.4`) is still security maintained by Canonical. Live kernel patches are available for this series until the security coverage window ends.

- **✓ kernel {kernel-version} is covered by Livepatch until {date}, please install available kernel updates and reboot before then**

  This specific kernel version (e.g. `5.4.0-216.236-generic`) is security maintained until the given date (its SRU end-of-security coverage date). You must install updates and reboot before then to stay covered.

- **✗ kernel is not covered by Livepatch**

  The running kernel version is not security maintained by Canonical Livepatch. Consider upgrading to a security maintained kernel.

- **✗ kernel is no longer covered by Livepatch**

  This kernel has reached end of life and no longer receives live kernel patches. Upgrade to a newer kernel.

- **✗ Livepatch coverage has ended; please upgrade the kernel and reboot**

  Security maintenance has ended and no SRU date was provided. An upgrade is required.

- **✗ Livepatch coverage ended {date}; please upgrade the kernel and reboot**

  Security maintenance has ended as of the specified date. Upgrade to continue receiving patches.

- **✗ unable to determine kernel support status; please contact Canonical support**

  An unexpected error occurred. Please [file a bug](https://bugs.launchpad.net/canonical-livepatch-client/+filebug) or contact Canonical support.

The `patch state` line can also have one of several values:

- **⧗ live kernel patches are downloaded, but the kernel module is not yet inserted**

  A patch has been downloaded but the kernel module has not yet been inserted.

- **⧗ patching the kernel**

  A patch is currently being applied.

- **✓ no live kernel patches available for kernel {kernel-version}**

  No patches exist yet for the kernel with the specified version.

- **✓ all applicable live kernel patch modules applied**

  All available live kernel patch modules for this kernel have been inserted and applied.

- **✗ Livepatch kernel module inserted but kernel bug detected**

  The kernel reported an error after the Livepatch kernel module was inserted.

- **✗ detected a crash last time the Livepatch kernel module was applied, check system logs with** `journalctl -f -u snap.canonical-livepatch.canonical-livepatchd`

  An earlier patch attempt caused a crash.

- **✗ unknown error occurred, please check system logs with** `journalctl -f -u snap.canonical-livepatch.canonical-livepatchd`

  An unexpected error occurred.

- **✗ kernel {kernel-version} contains a vulnerability that cannot be remediated with live kernel patching, please upgrade and reboot**

  This kernel has an unpatchable vulnerability. An upgrade is required.

- **✓ kernel upgraded after a reboot**

  The kernel was recently upgraded and rebooted successfully.

- **✗ failed to verify the signature of the Livepatch kernel module**

  Signature verification failed. The patch was not applied.

- **✗ failed to extract information about the Livepatch kernel module**

  Patch metadata could not be read.

- **✗ failed to load certificate used to verify the signature of the Livepatch kernel module**

  The required certificate could not be loaded.

- **✗ failed to confirm that the Livepatch kernel module was applied successfully**

  The client could not confirm that the Livepatch kernel module was applied successfully.

If live kernel patches have been applied, you will see a `patch version` field `patch version: 113.1` as in the status output above.

Patch versions map directly to [Ubuntu Livepatch Security Notices (LSN)](https://ubuntu.com/security/notices) published in the format `LSN-<version>`. The notices describe the vulnerabilities that were resolved in that version.

You can retrieve the corresponding LSN by mapping the version to the LSN identifier. For example:

- `patch version: 113.1` maps to [LSN-0113-1](https://ubuntu.com/security/notices/LSN-0113-1)
- `patch version: 94.1` maps to [LSN-0094-1](https://ubuntu.com/security/notices/LSN-0094-1)

**Note**: LSN identifiers are zero-padded (e.g. `113.1` maps to `LSN-0113-1`).

The canonical-livepatch status command accepts flags that change the output format:

- `--summary` shows a concise summary (similar to default output).

- `--verbose` shows extended details such as client version, architecture, boot time, and applied CVEs.

- `--show-secrets` shows sensitive machine tokens (requires sudo).

- `--format <format>` chooses the output format. Supported values:

  - `humane` (default, human-readable with minimal detail)

  - `json` (machine-readable JSON with extra metadata like architecture, CPU model, boot time, uptime, patch tier, etc.)

  - `yaml` (machine-readable YAML with extra metadata similar to JSON output)
