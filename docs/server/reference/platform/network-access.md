---
myst:
  html_meta:
    description: "Network access requirements for configured Livepatch Server frontends, upstreams, storage, databases, and monitoring, including connected and air-gapped deployments."
---

(server-reference-livepatch-on-prem-network-access-requirements)=

# Livepatch on-prem network access requirements

| Canonical-hosted destination | Port | When required |
| ---------------------------- | ---- | ------------- |
| `livepatch.canonical.com` | TCP 443 | Synchronizing from Canonical's hosted Livepatch service. |
| `livepatch-files.canonical.com` | TCP 443 | Downloading patch payloads supplied by Canonical's hosted service. |
| `contracts.canonical.com` | TCP 443 | Obtaining a resource token for upstream synchronization, and validating client resource tokens when the contracts service is enabled. |

Clients register with and send requests to the on-premises server, which validates auth and machine tokens locally. An air-gapped deployment can use {ref}`offline patch imports <server-how-to-guides-distribute-patches-offline>`, {ref}`authorization tokens issued by the on-premises server <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`, and an on-premises Ubuntu Pro service if required, with database, storage, and observability services kept inside the isolated environment. Depending on your configured integrations, also allow the following connections, using the actual hostnames and ports from your deployment:

| Caller | Destination | Port and access scope |
| ------ | ----------- | --------------------- |
| Clients, administrators, and downstream servers | Livepatch frontend, and any separate patch-download or upload endpoint | Configured listener, normally TCP 443 for HTTPS. Restrict administrative routes to authorized administration networks. |
| Ingress or reverse proxy | Livepatch HTTP backend | Configured backend port on a trusted or network-protected path, so callers cannot bypass the frontend. |
| Livepatch Server | PostgreSQL, upstream API and patch-download hosts, remote patch store, and optional Ubuntu Pro service | Configured ports, commonly TCP 5432 for PostgreSQL and TCP 443 for HTTPS. For chaining, upstream hosts can be internal rather than Canonical services. |
| Monitoring and log collectors | Livepatch `/metrics` endpoint and configured log destinations | Restrict `/metrics` to authorized monitoring infrastructure. For other ports, see the {ref}`logging and monitoring guide <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`. |

For snap installations, see the [snap network requirements](https://forum.snapcraft.io/t/network-requirements/5147).