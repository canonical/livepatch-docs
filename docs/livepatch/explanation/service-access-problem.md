---
myst:
  html_meta:
    description: "Service access problem - learn about this topic in Livepatch client."
---


(livepatch-explanation-i-cannot-access-the-livepatch-service)=

# I cannot access the Livepatch service

Network access to the Ubuntu Livepatch Service ([https://livepatch.canonical.com:443](https://livepatch.canonical.com/)) is necessary as well as snapd version 2.15 - check your snapd version with `snap version`.

In addition to that, sometimes it could be a problem with the proxy configurations. Please check out [this](/livepatch/how-to-guides/configure-proxy.md) guide on how to configure the Livepatch client to use a proxy.
