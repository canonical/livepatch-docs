---
myst:
  html_meta:
    description: "Network access requirements for configured Livepatch Server frontends, upstreams, storage, databases, and monitoring, including connected and air-gapped deployments."
---

(server-reference-livepatch-on-prem-network-access-requirements)=

# Livepatch on-prem network access requirements

Network requirements depend on the configured upstream, authentication method, storage, and monitoring integrations. Use the actual hostnames and ports from your deployment when defining firewall rules.

## Canonical-hosted services

| Destination | Port | When required |
| ----------- | ---- | ------------- |
| `livepatch.canonical.com` | TCP 443 | Synchronizing from Canonical's hosted Livepatch service. |
| `livepatch-files.canonical.com` | TCP 443 | Downloading patch payloads supplied by Canonical's hosted service. |
| `contracts.canonical.com` | TCP 443 | Connected patch-upstream tooling obtaining an entitlement credential. It is not used for local client authentication. |

Clients register with and send subsequent requests to the on-premises Livepatch Server. The server validates locally issued auth tokens and machine tokens without forwarding client authentication to an upstream service.

## Deployment-specific connections

| Caller | Destination | Port and access scope |
| ------ | ----------- | --------------------- |
| Clients, administrators, and downstream servers | Configured Livepatch frontend | Configured listener, normally TCP 443 for HTTPS. Restrict administrative routes to authorized administration networks. |
| Ingress or reverse proxy | Livepatch HTTP backend | Configured backend port on a trusted or network-protected path. Restrict direct access so callers cannot bypass the frontend. |
| Livepatch Server | Application PostgreSQL database and any separate PostgreSQL patch database | Configured PostgreSQL port, commonly TCP 5432, restricted to the workload and authorized database administrators. |
| Livepatch Server | Configured upstream API and patch-download hosts | Ports from the configured URLs, normally TCP 443 for HTTPS. For chaining, these can be internal hosts rather than Canonical services. |
| Livepatch Server | Configured remote patch store | Storage API and authentication endpoints required by the selected backend. Include Swift service-catalog endpoints where applicable. Filesystem storage does not require a separate storage-service connection. |
| Clients or the administrator tool | Separate patch-download or upload endpoint | Required when patches are transferred directly to or from storage rather than through the Livepatch API. Use the endpoint's configured port and access policy. |
| Livepatch Server | Optional Ubuntu Pro service | Configured service URL and port. A local service can replace the public endpoint for deployments using that integration. |
| Monitoring collectors | Livepatch `/metrics` endpoint | Server listener or protected proxy route, accessible only to authorized monitoring infrastructure. |
| Log collectors and observability integrations | Configured monitoring and log destinations | Ports and direction required by the selected integration. See the {ref}`logging and monitoring guide <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`. |

Additional enabled services can introduce further destinations. Check their configured endpoints rather than treating the Canonical-hosted list as a complete allowlist for every deployment.

## Air-gapped operation

An air-gapped deployment can use offline patch imports, authorization tokens issued by the on-premises server, and locally validated machine tokens. It can also use an on-premises Ubuntu Pro service when that integration is required. Keep database, storage, observability, and entitlement services inside the isolated environment.

See {ref}`offline patch distribution <server-how-to-guides-distribute-patches-offline>` and {ref}`client registration <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`.

## Installation and server updates

Charm, snap, and workload-image installation or updates have their own artifact-store requirements, separate from runtime patch synchronization. For snap installations, see the [snap network requirements](https://forum.snapcraft.io/t/network-requirements/5147). Permit the artifact sources required by your deployment platform, or arrange approved offline delivery.

For integration procedures, see {ref}`Integrate Livepatch Server securely with other services <server-how-to-guides-integrate-services-securely>`.