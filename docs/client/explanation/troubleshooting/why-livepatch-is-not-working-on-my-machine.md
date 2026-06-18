---
myst:
  html_meta:
    description: "Diagnose why Livepatch is not working, including unsupported kernels, HWE vs GA kernel differences, and Secure Boot key enrollment requirements."
---

(client-explanation-why-isnt-livepatch-working-on-my-machine)=

# Livepatch not working

This document explains common reasons why Livepatch may not function as expected and how to diagnose each situation.

## Unsupported kernels

Livepatch supports only kernels that have been released by the kernel team to the updates pocket — officially released kernels acquired through APT using Canonical's repository for system updates, or snap-based kernels released by Canonical to stable snap channels.

While a live kernel patch might successfully apply to a kernel from other sources, only certain kernels released by Canonical are supported. Kernels from the following sources are not supported:

- Kernels acquired from the development (proposed) kernel PPA
- Kernels acquired from the kernel team's build PPA
- Test kernels acquired from the kernel team's development PPAs
- Personally rebuilt kernels using the source Debian package
- Personally rebuilt kernels using Snapcraft
- Kernels acquired from an Ubuntu-derived distribution

Building a kernel with the same version markings as an officially supported kernel and attempting to load a Canonical-generated live patch into it will likely not work, and can potentially crash your system or corrupt your data.

Kernels running unsigned, out-of-tree drivers are tainted, and the kernel will refuse to apply live kernel patches in this state.

A full list of supported kernels is available in the [Supported kernels reference](/client/reference/platform/supported-kernels.md).

## HWE vs GA kernels

Livepatch support differs between Hardware Enablement (HWE) and General Availability (GA) kernels.

Newer releases of LTS Ubuntu Desktop default to the HWE kernel, meaning your machine runs a kernel version that updates alongside each Ubuntu release before settling on the kernel released with the next LTS. Ubuntu Server installations remain on the GA kernel. For more details on HWE kernels, see the [Canonical blog](https://canonical.com/blog/canonical-livepatch-gets-even-better-now-supporting-hardware-enablement-kernels) and [Ubuntu kernel variants page](https://ubuntu.com/kernel/variants).

Livepatch supports the GA kernel and the HWE kernel that you settle on with the next LTS. It does not fully support the interim kernels released every 6 months. This is why a machine running an LTS Ubuntu Desktop release may still display Livepatch messaging indicating that the kernel is not supported.

You can switch from an HWE kernel to the GA kernel by following the [LTS Enablement Stack instructions](https://wiki.ubuntu.com/Kernel/LTSEnablementStack). Back up your data and important information before making system-level changes.

Prior to Ubuntu 22.04 LTS, Livepatch offered no support for interim kernel versions. Livepatch has since added support for some flavours of interim HWE kernels. Desktop user kernels are the "generic" flavour, while public cloud kernels have their own unique flavours supporting cloud-specific functionality. Livepatch is now supported on interim HWE kernels for various public cloud flavours. Check your kernel flavour with `uname -r`.

A full list of GA and HWE supported kernels is available in the [Supported kernels reference](/client/reference/platform/supported-kernels.md).

## Secure Boot

If you are using Secure Boot with a kernel older than April 1, 2021, you must import the Livepatch public keys into your keyring. For newer kernels, the key is already included, and no import is necessary (though importing will not cause harm).

To import the Livepatch key:

```
sudo mokutil --import /snap/canonical-livepatch/current/keys/livepatch-kmod.x509
```

Enter a password if prompted by MOK, then reboot. Your BIOS will guide you through enrolling the new key in MOK. After enrollment, you can verify the module signatures.