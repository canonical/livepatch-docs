---
myst:
  html_meta:
    description: "Harden a Livepatch Server deployment with TLS frontends, protected storage, secure credentials, request limits, and restricted network and administrative access."
---

(server-how-to-guides-harden-your-livepatch-server-deployment)=

# Harden a Livepatch Server deployment

Use this guide to harden a Livepatch Server deployed with the machine charm, Kubernetes charm, or standalone snap. Start with the {ref}`secure configuration checklist <server-how-to-guides-configure-livepatch-server-securely>` before exposing a new deployment. The Juju examples assume that the Livepatch application is named `livepatch`.

## Enable TLS termination

The Livepatch Server does not natively terminate TLS. In on-premises deployments where clients and the server are on a trusted internal network, TLS between them is optional. However, if the server is accessible from untrusted networks or admin API endpoints are exposed, a TLS-terminating reverse proxy, Kubernetes ingress controller, or load balancer should be placed in front of the server.

| Deployment | TLS frontend |
| ---------- | ------------ |
| Kubernetes charm | An ingress provider using the `ingress` integration, or `nginx-route` for existing legacy integrations. |
| Machine charm | HAProxy connected through the Livepatch charm's `website` relation. |
| Standalone snap | A separately configured reverse proxy or load balancer, such as Nginx or HAProxy. |

For step-by-step TLS setup instructions, see the [TLS configuration guide](/server/how-to-guides/security/setup-tls.md).

In [federated deployments](/server/how-to-guides/patch-management/chain-livepatch-servers.md) where multiple on-premises servers are chained, TLS should also be enabled between them to protect the exchange of tier and patch data. Configure TLS termination on the upstream server just as for client-facing traffic.

## Protect database access

Keep PostgreSQL on a trusted, isolated network, restricted to the Livepatch application and authorized database administrators, and use a dedicated database and account. For a standalone snap with a remote database, set `sslmode=verify-full` in the connection string through the [snap configuration interface](/server/reference/platform/configuration.md#snap), use a hostname that matches the database certificate, and provide the issuing CA through `sslrootcert` at a path readable by the confined snap. For the charms, use these `postgresql_client` relation endpoints rather than `database-legacy`, and manage relation-provided credentials through Juju:

| Livepatch charm | Livepatch relation endpoint |
| --------------- | --------------------------- |
| Kubernetes | `postgresql` |
| Machine | `database` |

## Protect patch storage

| Backend | Controls |
| ------- | -------- |
| Filesystem | Restrict the patch directory to the workload and authorized administrators, and use host or volume encryption where required. |
| PostgreSQL | Apply the database access controls above to the patch database as well. |
| Amazon S3 | Set `patch-storage.s3-secure=true` to use HTTPS, and scope credentials and bucket permissions to the required operations. |
| OpenStack Swift | Use HTTPS for the authentication URL and the service-catalog endpoints, and scope access to the intended account and container. |
| Google Cloud Storage | Use HTTPS endpoints, restrict the service identity to the intended bucket, and protect any service-account key material. |
| Azure Blob Storage | Use HTTPS endpoints, including in custom connection strings, and scope credentials to the required resources. |

Do not disable certificate verification to work around trust failures, and check access from clients if they download patches from a separate endpoint. For the charms, prefer secret-backed options over credentials in Juju configuration; see the [configuration reference](/server/reference/platform/configuration.md). Follow {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>` for encryption at rest, backups, and storage migration.

## Configure a private CA for an optional Ubuntu Pro service

This applies only if you validate Ubuntu Pro resource tokens against a service with a private CA. It is not required for the local auth-token and machine-token workflow. For either charm, set `contracts.ca` to the base64-encoded PEM certificate with `juju config livepatch contracts.ca="$(base64 -w0 ca-cert.pem)"`, or use `contracts.ca: include-base64://path/to/ca-cert.pem` in a Juju bundle. Keep the service URL consistent with the certificate's hostname.

## Use the latest version

Keep the charm or standalone snap on the current `latest/stable` revision, as described in {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>`. Update charm-managed deployments through Juju, and do not manually refresh or unhold the snap managed by the machine charm.

## Secure admin credentials

Use individual administrator accounts and strong passwords. Generate each bcrypt username-and-hash pair with `htpasswd -nBC 10 admin` from the `apache2-utils` package, replacing `admin` with the username and entering the password when prompted. Treat the hash as sensitive, because it can be used for offline password guessing. For the standalone snap, set `auth.basic.enabled` and `auth.basic.users` with the `lp.` prefix, as described in the [administration tool setup guide](/server/how-to-guides/security/setup-administration-tool.md#password-authentication).

For the charms, store the complete `htpasswd` output as `value: 'admin:<bcrypt-hash>'` in a file named `admin-users.secret.yaml`, with `0600` permissions and outside version control. Separate multiple pairs with commas in the same value. Create and grant the secret, then enable it with the secret URI returned by `juju add-secret`. The secret takes precedence over `auth.basic.users`; after verifying administrator access, reset that obsolete value. Delete the secret file when finished.

```shell
juju add-secret livepatch-admin-users --file admin-users.secret.yaml
juju grant-secret livepatch-admin-users livepatch
juju config livepatch auth.basic.enabled=true auth.basic.users-secret='<secret-uri>'
juju config livepatch --reset auth.basic.users
```

## Configure request limits

`server.concurrency-limit` is the maximum number of requests processed at once, and `server.burst-limit` is the maximum number of requests processed or queued. A queue exists only when `server.burst-limit` is greater than `server.concurrency-limit`. Queued requests wait at most 10 seconds for a free slot. Requests beyond the limits, or that time out in the queue, are rejected with HTTP 503. The defaults are:

| Deployment | `server.concurrency-limit` | `server.burst-limit` | Effect |
| ---------- | -------------------------- | -------------------- | ------ |
| Machine and Kubernetes charms | 1000 | 500 | No queue. Requests over 1000 are rejected immediately. |
| Standalone snap | 200 | 500 | Up to 300 requests queue. |

Tune the values for your workload and resources, and monitor HTTP 503 responses using [logging and monitoring](/server/how-to-guides/operations/configure-logging-and-monitoring.md). These are concurrency limits, not rate limits. Configure connection limits, rate limits, and timeouts at your ingress or reverse proxy as well. For example, to process up to 200 requests and queue up to 300 more on either charm:

```shell
juju config livepatch server.concurrency-limit=200 server.burst-limit=500
```

For the standalone snap:

```shell
sudo snap set canonical-livepatch-server lp.server.concurrency-limit=200 lp.server.burst-limit=500
```

## Restrict network access

In production deployments:

* Admin API endpoints should only be accessible from trusted networks.
* The `/metrics` endpoint (Prometheus) should be restricted to monitoring infrastructure.
* Database ports should not be exposed to the public network.
* The HTTP backend and storage-management endpoints should only be reachable by authorized callers, such as the frontend, and outbound access should be limited to configured dependencies. Keep the proxy-to-server hop on a trusted network, because frontend TLS does not encrypt it.

## Maintain strict privileged access

For Juju-managed deployments, restrict controller, model, configuration, secret, and action access to authorized operators, because relation data and secrets can contain credentials. For the machine charm, also restrict SSH, `sudo`, and Juju execution access to the managed machines, and manage the workload through the charm rather than changing its snap or generated configuration manually. For the Kubernetes charm, also restrict pod execution, and use a network plugin that enforces `NetworkPolicy` resources.

### Snap

The snap uses `strict` confinement, restricting the server to only the `network` and `network-bind` interfaces.

All server data is stored under the `$SNAP_COMMON` and `$SNAP_DATA` directories. Maintain strict privileged access controls on the host machine to prevent unauthorized access to this data.

### Client machines

When a Livepatch Client is registered with the on-premises server, an authentication token is stored on the client machine. Ensure only authorized users have `sudo` access to hosts running the Livepatch Client, as the token grants the ability to authenticate against the server. For details on client registration, see the [client setup guide](/server/how-to-guides/operations/use-livepatch-client-with-on-prem-server.md).

### Charm (Kubernetes)

The charm deploys the server as a container. The server listens on port 8080 internally. Apply Kubernetes security best practices:

* Use `NetworkPolicy` resources to restrict pod-to-pod communication.
* Use Kubernetes RBAC to limit access to the server's namespace and secrets.

## Reset to original state

Use the secure configuration guide to choose between a {ref}`configuration-only reset <server-how-to-guides-reset-livepatch-server-configuration>`, which preserves stored data, and a {ref}`full reset <server-how-to-guides-fully-reset-livepatch-server>`, which removes application data before a fresh deployment. Removing and redeploying the application alone does not clear an external database or patch store.
