---
myst:
  html_meta:
    description: "Reference for Livepatch on-premises network access requirements, listing the hostnames and ports needed for outbound connectivity from firewalled deployments."
---

(server-reference-livepatch-on-prem-network-access-requirements)=

# Livepatch on-prem network access requirements

On firewalled deployments, the Livepatch on-premises server requires outbound access to the following hostnames:

- `livepatch.canonical.com`, port 443
- `livepatch-files.canonical.com`, port 443
- `contracts.canonical.com`, port 443

For snap installations, see the [snap network requirements](https://forum.snapcraft.io/t/network-requirements/5147).