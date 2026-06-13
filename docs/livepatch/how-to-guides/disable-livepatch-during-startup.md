---
myst:
  html_meta:
    description: "How to disable livepatch during startup with Livepatch client."
---


(livepatch-how-to-guides-how-to-disable-livepatch-during-startup)=

# How to disable Livepatch during startup

The Livepatch client will make a best-effort attempt to [prevent re-inserting and reloading a faulty patch](/livepatch/reference/patch-installation.md) that causes a system crash or causes kernel bug log entries. In the extremely rare case that a faulty patch causes a system crash loop, recovery methods are available to disable the Livepatch daemon, thereby preventing the application of the faulty patch. The following methods explain how to recover the system in such scenarios. These recovery methods rely on setting different values for the [Livepatch mode](/livepatch/how-to-guides/disable-client.md).

## Various ways to disable Livepatch

1. **Using the kernel command-line**

- If the faulty patch is causing a system crash and reboot loop, enter the GRUB boot menu during the boot process.
- Edit the boot options for the kernel you are booting into and add `canonical_livepatch_mode=stop` to the kernel command-line, to prevent the Livepatch daemon from starting up.
- Alternatively, you can add `canonical_livepatch_mode=no-apply` to the kernel command-line. This mode enables the Livepatch client and daemon and refreshes the patch information, but the patch is never applied to the kernel.
- The limitation of solely relying on this recovery method is that the Livepatch mode is not permanent. On a reboot, the Livepatch mode reverts to `normal`. This will start up the Livepatch daemon and apply the faulty patch to the kernel.

2. **Using the `/var/local/canonical_livepatch_mode` file**

- To ensure that the Livepatch mode persists across reboots and prevents patch application, the mode string can be written to the `/var/local/canonical_livepatch_mode` file. For example, to stop the Livepatch daemon from starting up on system initialization, the Livepatch mode `stop` can be written to the file as follows:

```shell
sudo bash -c 'echo -n stop > /var/local/canonical_livepatch_mode'
```

- The Livepatch mode can also be written to the `/var/local/canonical_livepatch_mode` file during the boot process, to disable the Livepatch client and daemon.
  - Boot into recovery mode for the kernel, get access to the root shell and execute the command shown above. Once the Livepatch mode has been written to the file, continue the normal boot process.
  - Boot into rescue mode by adding `systemd.unit=rescue.target` in the kernel command-line, which gives full access to the root user and local file systems. On getting access to the shell as the root user, execute the command shown above and then continue the normal boot process.

## Resume the normal operation of Livepatch

After recovering from the crash loop, resuming the normal operation of the Livepatch client and daemon is necessary to secure the running kernel. However, we need to ensure that enabling Livepatch does not result in a crash loop again.

- The Livepatch client sends anonymous pings before, during and after a patch is applied, to the Livepatch server. The client pings are continuously [monitored by the Livepatch team](/livepatch/reference/patch-lifecycle.md) to ensure the sanity of the patches. If the analysis of the ping metrics points towards a patch being faulty, it is [blocklisted](/livepatch/reference/patch-lifecycle.md) to prevent it from being served to client machines.
- Users can also [report bugs](https://bugs.launchpad.net/canonical-livepatch-client) upon facing a system crash loop that could be caused by the application of a patch. If the patch is found to be faulty by the Livepatch team, it will be blocklisted.
- After the faulty patch is blocklisted or a new and stable patch is released, it is safe to resume the normal operation of the Livepatch client.

The process to resume the normal operation of Livepatch depends on the recovery method used.

- When the kernel command-line parameter `canonical_livepatch_mode=<mode>` is used for recovery, the only way to resume normal operation is to reboot the system.
- If recovery is done by writing a Livepatch mode into the `/var/local/canonical_livepatch_mode` file, the first step to resume normal operation would be to change the mode to `normal` or delete the file. Then, run the command `sudo snap restart canonical-livepatch`.

## Enable Livepatch with a cutoff-date to withhold one or more LSNs

The process of reporting, blocklisting, and releasing a replacement Livepatch takes time. If installing a kernel package update is not possible, it is possible to exclude one or more LSNs by enabling Livepatch with a cutoff date. The `cutoff-date` configuration option available in the Livepatch client can be used to selectively apply some LSNs to a kernel, and selectively omit one or more newer LSNs. The `cutoff-date` config option allows users to set a date in the past, and only patches released before this `cutoff-date` will be applied to the kernel. This will allow users to apply the desired LSNs to the kernel, and omit certain LSNs by date. The following steps outline how this can be accomplished:

1. If the current Livepatch mode in the `/var/local/canonical_livepatch_mode` file is `stop` and the Livepatch daemon is not running, we cannot make any configuration changes.
2. Set the Livepatch mode to `no-apply` using `sudo bash -c 'echo -n no-apply > /var/local/canonical_livepatch_mode'`
3. Run `sudo snap restart canonical-livepatch`. This should enable the Livepatch daemon. The daemon will refresh patch information but never apply the patch to the kernel in this mode.
4. Once the daemon is enabled, set the `cutoff-date` configuration option to a date before the faulty patch was released. For example, `sudo canonical-livepatch config cutoff-date=2024-01-01T01:00:00Z`. The release date for the problematic LSN can be found [here](https://ubuntu.com/security/notices?details=lsn).
5. Set the Livepatch mode to `normal` or delete the Livepatch mode file.
6. Run `sudo snap restart canonical-livepatch`. This should enable the full operation of the Livepatch daemon. The daemon will get the most recent stable patch version due to the `cutoff-date` config option and apply it to the running kernel.

This process improves kernel security until a new and stable Livepatch is released for the kernel, at which point the cutoff-date configuration option can be disabled or changed to get the most recent Livepatch.
