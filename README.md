# Livepatch Documentation

Canonical Livepatch patches high and critical Linux kernel vulnerabilities without requiring a system reboot, shrinking the window between the publication of a security fix and its application on running machines. Livepatch is part of the [Ubuntu Pro](https://ubuntu.com/pro) offering.

The Ubuntu Livepatch offering consists of the Livepatch Client, the Livepatch service hosted by Canonical, and an optional on-premises Livepatch Server. The client runs on each registered machine, periodically checks for available patches, and downloads, verifies, and installs them onto the running kernel.

Livepatch is built for critical infrastructure where unscheduled downtime is not acceptable. By applying live kernel patches for high and critical kernel vulnerabilities, upgrades can be scheduled at a convenient time without interrupting services.

Livepatch is used by system administrators, security teams, and platform operators who manage Ubuntu systems at scale. The on-premises server is designed for customers operating machines in network-restricted environments where connecting client instances to external servers is not permitted. It supports staged rollout policies, centralised patch distribution, and integration with air-gapped environments.

## Support

Support is available to customers with an [Ubuntu Pro](https://ubuntu.com/pro) subscription through the [Canonical support portal](https://portal.support.canonical.com/). See the {ref}`support <support>` page for more details.

## In this documentation

|                    |                                                                     |
|--------------------|---------------------------------------------------------------------|
| **Client** | {ref}`How-to guides <client-how-to-guides>` • {ref}`Reference <client-reference>` • {ref}`Explanation <client-explanation>` |
| **Server** | {ref}`Tutorial <server-tutorial>` • {ref}`How-to guides <server-how-to-guides>` • {ref}`Reference <server-reference>` • {ref}`Explanation <server-explanation>` |
| **Releases** | {ref}`Release notes for Livepatch Client and Server <release-notes>` |
| **Resources** | {ref}`Contribute to the documentation <contribute-to-docs>` • {ref}`Get support and report bugs <support>` |

## How the documentation is organized

This documentation uses the [Diátaxis documentation structure](https://diataxis.fr/).

- The {ref}`server-tutorial` takes you step-by-step through deploying the Livepatch on-premises server for the first time, in environments such as LXD, MicroK8s, and air-gapped networks. <br>
- {ref}`How-to guides <server-how-to-guides>` assume you have basic familiarity with Livepatch. They provide step-by-step instructions for achieving a practical goal, from enabling the client on a machine to deploying the on-premises server and managing patches. <br>
- {ref}`Reference <server-reference>` contains technical specifications for the client and server — including supported platforms, networking requirements, authentication mechanisms, patch storage, and telemetry. <br>
- {ref}`Explanation <server-explanation>` discusses background topics such as kernel live patching, the patch lifecycle, security models, and troubleshooting — helping you build a deeper understanding of how Livepatch works. <br>

## Project and community

Livepatch is a member of the Ubuntu family. It welcomes community contributions, suggestions, fixes, and constructive feedback.

### Get involved

* [Get support](https://ubuntu.com/support/community-support)
* [Join the Discourse forum](https://discourse.ubuntu.com/c/project/livepatch/82)
* {ref}`Contribute to the documentation <contribute-to-docs>`

### Releases and policies

* {ref}`Release notes <release-notes>`
* [Our Code of Conduct](https://ubuntu.com/community/ethos/code-of-conduct)

### Commercial support

Thinking about using Livepatch for your next project? [Get in touch!](https://ubuntu.com/security/livepatch#get-in-touch)
