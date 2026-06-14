---
myst:
  html_meta:
    description: "Livepatch client documentation home."
---


(client)=

# Client

Livepatch client is the software running on a machine, that periodically checks for the availability of new patches. Once new patches are available, they are downloaded, verified and applied to the current kernel.

## In this documentation

|                                                                                              | |
|----------------------------------------------------------------------------------------------|-|
| Tutorials </br> Get started with Canonical Livepatch Client.                                 | [How-to guides](/client/how-to-guides/index.md) </br> Step-by-step guides covering key operations and common tasks</br> |
| [Explanation](/client/explanation/index.md) </br> Discussion and clarification of key topics | [Reference](/client/reference/index.md) </br> Technical information - specifications, APIs, architecture |

## Getting support

Canonical customers can receive support and report issues on the Ubuntu Livepatch service, Livepatch client or Livepatch on-prem, on Canonical's [support portal](https://portal.support.canonical.com/).

The projects maintain bug trackers at

- [Livepatch client bug tracker](https://bugs.launchpad.net/canonical-livepatch-client/+filebug)
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
