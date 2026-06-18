---
myst:
  html_meta:
    description: "How to disable Livepatch during startup."
---


(client-how-to-guides-how-to-disable-livepatch-during-startup)=

# How to disable Livepatch during startup

The Livepatch Client makes a best-effort attempt to [prevent re-inserting and reloading a faulty patch](/client/explanation/patches/patch-installation.md) that causes a system crash or kernel bug log entries. In the extremely rare case that a faulty patch causes a system crash loop, recovery methods are available to disable the Livepatch daemon, thereby preventing the application of the faulty patch. The following methods explain how to recover the system in such scenarios. These recovery methods rely on setting different values for the [Livepatch mode](/client/how-to-guides/installation/disable-client.md).

## Disable Livepatch using the kernel command line

1. If the faulty patch is causing a system crash and reboot loop, enter the GRUB boot menu during the boot process.
2. Edit the boot options for the kernel being booted and add `canonical_livepatch_mode=stop` to the kernel command line, to prevent the Livepatch daemon from starting.
3. Alternatively, add `canonical_livepatch_mode=no-apply` to the kernel command line. This mode enables the Livepatch Client and daemon and refreshes the patch information, but the patch is never applied to the kernel.

The limitation of relying solely on this recovery method is that the Livepatch mode is not permanent. On a reboot, the Livepatch mode reverts to `normal`. This starts the Livepatch daemon and applies the faulty patch to the kernel.

## Disable Livepatch using the mode file

To ensure that the Livepatch mode persists across reboots and prevents patch application, write the mode string to the `/var/local/canonical_livepatch_mode` file. For example, to stop the Livepatch daemon from starting on system initialization, write the Livepatch mode `stop` to the file:

```shell
sudo bash -c 'echo -n stop > /var/local/canonical_livepatch_mode'
```

The Livepatch mode can also be written to the `/var/local/canonical_livepatch_mode` file during the boot process, to disable the Livepatch Client and daemon:

* Boot into recovery mode for the kernel, get access to the root shell, and execute the command shown above. Once the Livepatch mode has been written to the file, continue the normal boot process.
* Boot into rescue mode by adding `systemd.unit=rescue.target` to the kernel command line, which gives full access to the root user and local file systems. On gaining access to the shell as the root user, execute the command shown above and then continue the normal boot process.

## Resume normal operation of Livepatch

After recovering from the crash loop, resume the normal operation of the Livepatch Client and daemon to secure the running kernel. Ensure that enabling Livepatch does not result in a crash loop again.

* The Livepatch Client sends anonymous pings before, during, and after a patch is applied to the Livepatch Server. The client pings are continuously [monitored by the Livepatch team](/client/explanation/patches/patch-lifecycle.md) to ensure the sanity of the patches. If the analysis of the ping metrics points towards a patch being faulty, it is [blocklisted](/client/explanation/patches/patch-lifecycle.md) to prevent it from being served to client machines.
* [Report bugs](https://bugs.launchpad.net/canonical-livepatch-client) upon encountering a system crash loop that could be caused by the application of a patch. If the patch is found to be faulty by the Livepatch team, it is blocklisted.
* After the faulty patch is blocklisted or a new and stable patch is released, it is safe to resume the normal operation of the Livepatch Client.

The process to resume normal operation depends on the recovery method used:

* When the kernel command line parameter `canonical_livepatch_mode=<mode>` is used for recovery, the only way to resume normal operation is to reboot the system.
* If recovery is done by writing a Livepatch mode into the `/var/local/canonical_livepatch_mode` file, change the mode to `normal` or delete the file. Then, run `sudo snap restart canonical-livepatch`.

## Enable Livepatch with a cutoff date to withhold one or more LSNs

The process of reporting, blocklisting, and releasing a replacement Livepatch takes time. If installing a kernel package update is not possible, exclude one or more LSNs by enabling Livepatch with a cutoff date. The `cutoff-date` configuration option available in the Livepatch Client can be used to selectively apply some LSNs to a kernel, and selectively omit one or more newer LSNs. The `cutoff-date` config option allows setting a date in the past, and only patches released before this `cutoff-date` are applied to the kernel. This allows applying the desired LSNs to the kernel, and omitting certain LSNs by date. The following steps outline how to accomplish this:

1. If the current Livepatch mode in the `/var/local/canonical_livepatch_mode` file is `stop` and the Livepatch daemon is not running, no configuration changes can be made.
2. Set the Livepatch mode to `no-apply` using `sudo bash -c 'echo -n no-apply > /var/local/canonical_livepatch_mode'`.
3. Run `sudo snap restart canonical-livepatch`. This enables the Livepatch daemon. The daemon refreshes patch information but never applies the patch to the kernel in this mode.
4. Once the daemon is enabled, set the `cutoff-date` configuration option to a date before the faulty patch was released. For example, `sudo canonical-livepatch config cutoff-date=2024-01-01T01:00:00Z`. The release date for the problematic LSN can be found on the [Ubuntu Security Notices page](https://ubuntu.com/security/notices?details=lsn).
5. Set the Livepatch mode to `normal` or delete the Livepatch mode file.
6. Run `sudo snap restart canonical-livepatch`. This enables the full operation of the Livepatch daemon. The daemon retrieves the most recent stable patch version due to the `cutoff-date` config option and applies it to the running kernel.

This process improves kernel security until a new and stable Livepatch is released for the kernel, at which point the `cutoff-date` configuration option can be disabled or changed to retrieve the most recent Livepatch.