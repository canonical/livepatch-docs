---
myst:
  html_meta:
    description: "What happens when a problem cannot be patched? - learn about this topic in Livepatch client."
---


(client-explanation-what-happens-when-a-problem-occurs-that-cant-be-patched)=

# What happens when a problem occurs that can’t be patched?

When an un-patchable security issue occurs, users *must* upgrade to a version of the kernel that is fixed, and reboot. Problems of this type are announced on the mailing list via LSN. Kernels prior to the levels named in that announcement will no longer receive live kernel patches.

The Livepatch client will report a state of "kernel-upgrade-required" if you are running a kernel that no longer receives live kernel patches due to an earlier un-patchable kernel security issue. This notice will appear in the user's desktop notifications, and in the text-based message of the day (MOTD) if logged into a terminal.
