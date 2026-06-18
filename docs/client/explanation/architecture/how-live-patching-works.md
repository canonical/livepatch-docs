---
myst:
  html_meta:
    description: "How kernel live patching works - learn about this topic in Livepatch client."
---


(client-explanation-how-kernel-live-patching-works)=

# How kernel live patching works

## The live kernel patching process

When a high or critical vulnerability is detected on the Linux kernel Canonical creates a live kernel patch addressing the vulnerability. After the live kernel patch is made available, it is tested in Canonical’s internal server farm, and then promoted gradually to a series of testing tiers ensuring that any released live kernel patch has been tested sufficient time on live systems. Once the patch is released a [Livepatch Security Notice](https://ubuntu.com/security/notices) is issued and systems that enable the Livepatch client will receive the patch over an authenticated channel and apply it.

## How does kernel live patching work?

There are many types of vulnerabilities and many reasons behind them, such as a logic error or a missing check in a small piece of code, and others. On the high level the live kernel patch will provide new kernel code replacing the vulnerable one, and will update the rest of the kernel to use the new code. The diagram below shows how a kernel vulnerability is being patched using Ubuntu Livepatch.

![Ubuntu Livepatch kernel patching diagram](/_static/images/at-a-glance-diagram.svg)

The simplistic description above shows the principle, but also hints on why some vulnerabilities that depend on very complex code interactions cannot be remediated with live kernel patching. When a kernel vulnerability cannot be remediated with live kernel patching, a [Livepatch Security Notice](https://ubuntu.com/security/notices) is issued that advises to apply any pending kernel updates and reboot.
