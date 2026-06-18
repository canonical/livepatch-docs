---
myst:
  html_meta:
    description: "Reference and how-to documentation for the Livepatch Client, including configuration, networking, patch management, and troubleshooting."
---

(client)=

# Client

The Livepatch Client runs on each registered machine to periodically check for the availability of new live kernel patches. When new patches are available, the client downloads, verifies, and applies them to the running kernel without requiring a system reboot.

The Livepatch Client can connect to Canonical's hosted Livepatch service or to an on-premises Livepatch Server.

## In this documentation

|     |     |
| --- | --- |
| [How-to guides](/client/how-to-guides/index.md) </br> Step-by-step guides covering key operations and common tasks. | [Reference](/client/reference/index.md) </br> Technical reference for the Livepatch Client, including platform support, networking requirements, and patch security. |
| [Explanation](/client/explanation/index.md) </br> Discussion and clarification of key Livepatch Client topics. | |

## Getting support

Canonical customers can receive support and report issues with the Ubuntu Livepatch service, the Livepatch Client, or Livepatch on-prem through the [Canonical support portal](https://portal.support.canonical.com/).

The projects maintain bug trackers at:

- [Livepatch Client bug tracker](https://bugs.launchpad.net/canonical-livepatch-client/+filebug)
- [Livepatch on-prem bug tracker](https://bugs.launchpad.net/livepatch-onprem/+filebug)

```{toctree}
:titlesonly:
:maxdepth: 2
:glob:
:hidden:

How-to guides <how-to-guides/index.md>
Reference <reference/index.md>
Explanation <explanation/index.md>
```