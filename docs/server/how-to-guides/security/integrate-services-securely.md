---
myst:
  html_meta:
    description: "Secure Livepatch Server connections to its frontend, upstream, database, patch storage, and monitoring services."
---

(server-how-to-guides-integrate-services-securely)=

# Integrate Livepatch Server securely with other services

Use this guide when connecting a Livepatch Server deployment to its supporting services.

## Connect supporting services

| Integration | Security guidance |
| ----------- | ----------------- |
| Ingress or reverse proxy | Terminate HTTPS at the frontend and restrict the HTTP backend to it; see {ref}`Set up TLS <server-how-to-guides-how-to-setup-tls-for-livepatch-on-prem>`. |
| PostgreSQL | Use the charm's database relation or the standalone snap's connection setting, following the [database access guidance](/server/how-to-guides/security/harden-your-deployment.md#protect-database-access). |
| Patch storage | Apply the [backend-specific controls](/server/how-to-guides/security/harden-your-deployment.md#protect-patch-storage), including for clients that transfer patches directly to or from storage. |
| Another Livepatch Server | Use the HTTPS URL of the upstream and its synchronization credential across untrusted networks; see {ref}`Chain on-premises servers <server-how-to-guides-chain-on-prem-servers>`. |
| Optional Ubuntu Pro service | Needed only for resource-token validation; for a private CA, follow the [CA configuration procedure](/server/how-to-guides/security/harden-your-deployment.md#configure-a-private-ca-for-an-optional-ubuntu-pro-service). |
| Monitoring and log collection | Restrict `/metrics` access and protect the receiving services and their credentials; see {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`. |

## Minimize interference with other services

To avoid saturating shared networks and services, use {ref}`patch sync filters <server-reference-patch-sync-filters>` to download only the architectures, kernel flavours, and versions your fleet needs, and set `patch-sync.interval` according to your patch-availability requirements. Configure the server's [request limits](/server/how-to-guides/security/harden-your-deployment.md#configure-request-limits), plus connection and rate limits at the frontend. Monitor rejected requests and synchronization failures through your monitoring integrations.

## Verify the integration

After configuration changes, check that administrator access and client registration work through the frontend, synchronization or offline import and client patch downloads succeed, the database and storage report no errors, monitoring receives data, and the {ref}`network rules <server-reference-livepatch-on-prem-network-access-requirements>` still block direct backend access.
