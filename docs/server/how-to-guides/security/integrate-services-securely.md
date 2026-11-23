---
myst:
  html_meta:
    description: "Secure Livepatch Server connections to its frontend, upstream, database, patch storage, and monitoring services."
---

(server-how-to-guides-integrate-services-securely)=

# Integrate Livepatch Server securely with other services

Use this guide when connecting a machine-charm, Kubernetes-charm, or standalone-snap deployment to its supporting services.

## Configure the required integrations

Choose the integrations needed by your deployment and follow their setup guides:

| Integration | Configuration and security guidance |
| ----------- | ----------------------------------- |
| Ingress or reverse proxy | Follow {ref}`Set up TLS <server-how-to-guides-how-to-setup-tls-for-livepatch-on-prem>`. Terminate HTTPS at the frontend when crossing untrusted networks and restrict the private HTTP backend to the frontend and other authorized callers. |
| PostgreSQL | Use the charm's database relation or the standalone snap's database connection setting. Follow the [database access guidance](/server/how-to-guides/security/harden-your-deployment.md#protect-database-access), including the standalone snap's database TLS procedure where applicable. Manage relation-provided credentials through Juju. |
| Patch storage | Apply the [backend-specific access and transport controls](/server/how-to-guides/security/harden-your-deployment.md#protect-patch-storage) for filesystem, PostgreSQL, Amazon S3, OpenStack Swift, Google Cloud Storage, or Azure Blob Storage. Check access from clients and administrator workstations too if they transfer patches directly to or from storage. |
| Another Livepatch Server | Follow {ref}`Chain on-premises servers <server-how-to-guides-chain-on-prem-servers>`. Configure the intended upstream URL and its synchronization credential, using HTTPS across untrusted networks. |
| Optional Ubuntu Pro service | Configure this integration if using Ubuntu Pro resource-token validation. For a service with a private CA, follow the [CA configuration procedure](/server/how-to-guides/security/harden-your-deployment.md#configure-a-private-ca-for-an-optional-ubuntu-pro-service). This is separate from the local auth-token and machine-token workflow. |
| Monitoring and log collection | Follow {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>` for the integrations supported by your deployment. Restrict `/metrics` access and protect the receiving services and their credentials. |

Use supported Juju or standalone snap configuration rather than editing generated workload configuration. For backups or a storage migration, follow {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`.

## Apply deployment-specific network rules

Use the {ref}`network-access reference <server-reference-livepatch-on-prem-network-access-requirements>` to identify the callers, destinations, and ports for your configured integrations. Restrict database and storage-management access separately from client-facing patch access. Keep administrative routes and monitoring endpoints limited to their intended callers.

Check both the upstream API and the patch-download destinations. A successful connection to the upstream API does not establish that the server can retrieve patches from a separate download host. The same applies to clients redirected to a separate patch endpoint.

For an air-gapped deployment, use {ref}`offline patch imports <server-how-to-guides-distribute-patches-offline>` and services reachable within the isolated environment. Register clients using {ref}`authorization tokens issued by the on-premises server <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`; subsequent requests use locally validated machine tokens.

## Control synchronization and request load

Use {ref}`patch sync filters <server-reference-patch-sync-filters>` to download the architectures, kernel flavours, and versions needed by your fleet. Set `patch-sync.interval` according to your patch-availability requirements.

Use the existing [request-limit guidance](/server/how-to-guides/security/harden-your-deployment.md#configure-request-limits) when configuring the frontend and server. Monitor rejected requests and synchronization failures through the {ref}`logging and monitoring integrations <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.

## Check the integration

After configuration changes:

1. Check administrator access and client registration through the intended frontend, including certificate trust when using HTTPS.
2. Verify an upstream synchronization or offline import, then retrieve a patch through the path used by clients, including any separate download endpoint.
3. Check database and storage errors and confirm that configured monitoring and log collection receive data.
4. Confirm that network restrictions still prevent direct backend access from unauthorized callers.
