---
myst:
  html_meta:
    description: "Understand how kernel live patching works in Livepatch, including the process from vulnerability detection to patch delivery and application."
---

(client-explanation-how-kernel-live-patching-works)=

# How kernel live patching works

When a high or critical vulnerability is detected in the Linux kernel, Canonical creates a live kernel patch to address it. After the patch is developed, it is tested in Canonical's internal server farm, then promoted gradually through a series of testing tiers to ensure it has been tested for a sufficient period on live systems. Once released, a [Livepatch Security Notice](https://ubuntu.com/security/notices) (LSN) is issued, and systems running the Livepatch Client receive and apply the patch over an authenticated channel.

## The live patching process

Kernel vulnerabilities can arise from many causes, such as logic errors or missing checks in small sections of code. At a high level, a live kernel patch provides new kernel code that replaces the vulnerable code and updates the rest of the kernel to use the new code.

![Ubuntu Livepatch kernel patching diagram](/_static/images/at-a-glance-diagram.svg)

The principle above also explains why some vulnerabilities that depend on complex code interactions cannot be fixed through live kernel patching. When a kernel vulnerability cannot be addressed with a live patch, a [Livepatch Security Notice](https://ubuntu.com/security/notices) is issued advising you to apply any pending kernel updates and reboot.
