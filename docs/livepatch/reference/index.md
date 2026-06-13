---
myst:
  html_meta:
    description: "Reference - technical reference for Livepatch client."
---


(livepatch-reference)=

# Reference

Technical information - security, APIs, architecture, etc., related to Livepatch.

## Networking

Livepatch client requires Internet access in order to fetch kernel patches from the server.

- [Network requirements](/livepatch/reference/network-requirements.md)

## Compatibility

Livepatch determines which kernel patch may be applied based on your kernel version.

- [Supported kernels](/livepatch/reference/supported-kernels.md)
- [How do I know if the patch is designed for my system?](/livepatch/reference/patch-installation.md#how-do-i-know-if-the-patch-is-designed-for-my-system)

## Security and privacy

Livepatch sends specific data about your system in order to patch your kernel.

- [Data sent](/livepatch/reference/data-sent.md)
- [Patch Security](/livepatch/reference/patch-security.md)

## Kernel patching

Livepatch inserts modules into a running kernel, this has inherent risks and the following can detail some of these risks and misunderstandings.

- [Patch Lifecycle](/livepatch/reference/patch-lifecycle.md)
- [Patch Installation](/livepatch/reference/patch-installation.md)

## Configuration

Livepatch client can be configured with a variety of options, which are listed here.

- [Configuration Options](/livepatch/reference/configuration-options.md)

```{toctree}
:titlesonly:
:maxdepth: 2
:glob:
:hidden:

Network requirements <network-requirements.md>
Data sent <data-sent.md>
Supported kernels <supported-kernels.md>
Patch lifecycle <patch-lifecycle.md>
Patch security <patch-security.md>
Patch installation <patch-installation.md>
Content caching <content-caching.md>
Configuration options <configuration-options.md>
```
