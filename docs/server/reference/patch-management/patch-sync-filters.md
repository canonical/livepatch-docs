---
myst:
  html_meta:
    description: "Reference for Livepatch on-premises patch sync filters, covering architecture, flavour, and minimum kernel version filtering to control which patches are synchronised."
---

(server-reference-patch-sync-filters)=

# Patch sync filters

Livepatch on-prem synchronises patches from an upstream server. This synchronisation process is configurable through filters that limit which patches are downloaded.

The primary synchronisation filters are:

- **System architecture** — Limit patches to specific CPU architectures.
- **Flavours** — Limit patches to specific kernel flavours.
- **Minimum kernel version** — Only synchronise patches for kernel versions greater than a specified minimum.

Configure sync filters to reduce the disk space consumed by patch files, or when client machines are limited to a specific set of kernel variants.

For a complete list of available sync filter options, see the [Livepatch Server configuration reference](/server/reference/platform/configuration.md).