---
myst:
  html_meta:
    description: "Diagnose Livepatch service access problems including network connectivity to the Livepatch service and proxy configuration requirements."
---

(client-explanation-i-cannot-access-the-livepatch-service)=

# Service access problems

Network access to the Ubuntu Livepatch Service at `https://livepatch.canonical.com:443` is required for the client to function. The system must also be running snapd version 2.15 or later. Check your snapd version with `snap version`.

If network access is available but the client cannot connect, the issue may be related to proxy configuration. See the guide on [configuring the Livepatch client to use a proxy](/client/how-to-guides/configuration/configure-proxy.md).