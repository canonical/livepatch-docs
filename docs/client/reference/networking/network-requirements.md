---
myst:
  html_meta:
    description: "Reference for Livepatch Client firewall requirements, listing the hostnames and ports needed for outbound connectivity."
---

(client-reference-livepatch-client-firewall-configuration)=

# Livepatch Client firewall configuration

On firewalled machines, the Livepatch Client requires outbound access to the following hostnames:

- `livepatch.canonical.com`, port 443
- `livepatch-files.canonical.com`, port 443

If the Livepatch Client is enabled through Ubuntu Pro, additional access to `contracts.canonical.com` on port 443 is required.

For snap installations, see the [snap network requirements](https://forum.snapcraft.io/t/network-requirements/5147).

```{note}
Before October 2023, patches were served over HTTP. All patch downloads now use HTTPS.
```