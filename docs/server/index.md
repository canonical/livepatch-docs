---
myst:
  html_meta:
    description: "Reference and how-to documentation for the Livepatch on-premises server, including deployment, configuration, authentication, patch management, and telemetry."
---

(server)=

# Server

Livepatch on-prem is an on-premises deployment of the Livepatch Server. It pulls live kernel patch updates from Canonical and allows fine-grained control over patch rollout to the machines in your infrastructure.

## In this documentation

|     |     |
| --- | --- |
| [Tutorial](/server/tutorial/index.md) </br> Get started with a hands-on introduction to deploying Livepatch. | [How-to guides](/server/how-to-guides/index.md) </br> Step-by-step guides covering key operations and common tasks. |
| [Reference](/server/reference/index.md) </br> Technical reference for the Livepatch Server, including platform requirements, authentication, and patch management. | [Explanation](/server/explanation/index.md) </br> Discussion and clarification of key Livepatch Server topics. |

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

Tutorial <tutorial/index.md>
How-to guides <how-to-guides/index.md>
Reference <reference/index.md>
Explanation <explanation/index.md>
```