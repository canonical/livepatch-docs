---
myst:
  html_meta:
    description: "Understand what happens when a kernel vulnerability cannot be patched via live kernel patching, including kernel upgrades, reboots, and client notifications."
---

(client-explanation-what-happens-when-a-problem-occurs-that-cant-be-patched)=

# Unpatchable problems

When a security issue occurs that cannot be addressed with a live kernel patch, you must upgrade to a fixed version of the kernel and reboot. Problems of this type are announced on the mailing list via LSNs. Kernels prior to the levels identified in that announcement will no longer receive live kernel patches.

The Livepatch client reports a state of "kernel-upgrade-required" if you are running a kernel that no longer receives live kernel patches due to an earlier unpatchable kernel security issue. This notice appears in the user's desktop notifications and in the text-based message of the day (MOTD) when logged into a terminal.