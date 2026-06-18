---
myst:
  html_meta:
    description: "Understand how Livepatch patch installation works, including module insertion, crash loop prevention, boot behavior, and kernel version matching."
---

(client-explanation-patches-patch-installation)=

# Patch installation

This document explains how live kernel patches are installed, how the Livepatch Client prevents crash loops, and how patches are matched to the running kernel.

## How patches are installed

When the Livepatch Client downloads a patch, it is stored on the machine's filesystem. The patch contains a Linux kernel module responsible for patching the running kernel. The client inserts this kernel module by making the appropriate syscall. The module affects only the in-memory kernel, without modifying the installed kernel on disk.

## Crash loop prevention

If a module causes a system crash, it is not loaded on reboot. The Livepatch Client makes a best-effort attempt to avoid re-inserting and reloading the patch if it detects a system crash within 10 seconds of loading the module.

If a patch contains a bug, the Livepatch Client performs a best-effort match against kernel logs to locate bug messages after insertion. When a bug is detected, the client reports that the module was inserted but caused a bug, and the patch is not reapplied on reboot.

In both crash and bug scenarios, a lockfile is created containing the bug trace. On the next reboot, if the lockfile is present, the patch is not inserted again. Once Canonical becomes aware that the patch is faulty, it is blocked from delivery.

## Boot behavior

Modules are not inserted at boot time. Patches are inserted when the Livepatch Client daemon service starts. This design prevents placing a system into a crash loop during startup.

## Kernel version matching

Each patch payload designed for Livepatch contains the kernel version in the format `ABI.BUILDNO-FLAVOUR`, which is matched against the currently running kernel. This ensures a patch is only applied to the kernel it was built for.
