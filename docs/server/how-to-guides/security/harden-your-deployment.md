---
myst:
  html_meta:
    description: "Harden a Livepatch Server deployment with TLS frontends, protected storage, secure credentials, request limits, and restricted network and administrative access."
---

(server-how-to-guides-harden-your-livepatch-server-deployment)=

# Harden a Livepatch Server deployment

Use this guide to harden a Livepatch Server deployed with the machine charm, Kubernetes charm, or standalone snap. Start with the {ref}`secure configuration checklist <server-how-to-guides-configure-livepatch-server-securely>` before exposing a new deployment.

The Juju examples assume that the Livepatch application is named `livepatch`. Replace it with your application name.

## Enable TLS termination

The Livepatch Server does not natively terminate TLS. TLS is optional within a fully trusted internal network, but recommended wherever clients, administrators, or chained servers communicate across untrusted networks. Without TLS, requests, responses, and authentication credentials travel in plaintext.

Choose the frontend for your deployment:

| Deployment | TLS frontend |
| ---------- | ------------ |
| Kubernetes charm | An ingress provider using the `ingress` integration for new deployments, or `nginx-route` for existing legacy integrations. |
| Machine charm | HAProxy connected through the Livepatch charm's `website` relation. |
| Standalone snap | A separately configured reverse proxy or load balancer, such as Nginx or HAProxy. |

Follow the [TLS configuration guide](/server/how-to-guides/security/setup-tls.md) to connect the frontend and configure certificate trust. Restrict direct access to the HTTP backend so that clients cannot bypass the frontend's access controls. Frontend TLS does not encrypt the separate proxy-to-server hop; keep that hop within a trusted network or protect it at the network layer.

For [chained servers](/server/how-to-guides/patch-management/chain-livepatch-servers.md), configure TLS termination on the upstream server and use its HTTPS address for synchronization across untrusted networks.

## Protect database access

Keep PostgreSQL on a trusted, isolated network. Restrict its port to the Livepatch application and authorized database administrators, and use a dedicated database and account rather than sharing a privileged account with unrelated services. Include database backups and snapshots in your access-control and encryption-at-rest policy.

For current charm deployments, obtain database access through the `postgresql_client` integration:

| Livepatch charm | Livepatch relation endpoint |
| --------------- | --------------------------- |
| Kubernetes | `postgresql` |
| Machine | `database` |

These are distinct from the `database-legacy` endpoint. Manage relation-provided credentials through Juju rather than manually changing a charm-managed workload.

### Standalone snap database TLS

For a standalone snap using a remote PostgreSQL database, set `sslmode=verify-full` in the database connection string. Use a database hostname that matches its certificate and provide the issuing CA through `sslrootcert`, using a path readable by the confined snap. Encryption without certificate and hostname validation does not establish that the server is the intended database.

Manage the connection string through the supported [snap configuration interface](/server/reference/platform/configuration.md#snap), not by editing generated configuration files.

## Protect patch storage

Apply the following controls to the backend you use:

| Backend | Controls |
| ------- | -------- |
| Filesystem | Restrict access to the patch directory to the workload and authorized administrators. Use host or volume encryption where required, including for backups. |
| PostgreSQL | Apply the database access controls above to the patch database as well. If it is a separate database, protect its credentials and backups separately. |
| Amazon S3 | Set `patch-storage.s3-secure=true` to use HTTPS. Scope credentials and bucket permissions to the required patch operations. |
| OpenStack Swift | Use an HTTPS authentication URL and ensure the object-storage endpoints returned by the service catalog also use HTTPS. Scope access to the intended account and container. |
| Google Cloud Storage | Use HTTPS endpoints and restrict the service identity's permissions to the intended bucket and required operations. Protect any service-account key material. |
| Azure Blob Storage | Use HTTPS endpoints, including in custom connection strings. Scope the configured identity or credentials to the required storage resources and operations. |

Use certificate validation for HTTPS connections. Do not work around trust failures by disabling verification. Check access from both the server and clients if your deployment redirects clients to a separate patch-download endpoint.

Configure storage-provider encryption at rest and protected backups according to your requirements. For the charms, prefer the backend's supported secret-backed configuration options over credentials stored directly in Juju configuration. See the [configuration reference](/server/reference/platform/configuration.md) and the configuration page for your charm.

Follow {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>` for consistent backups, protected database exports and restores, patch-storage migration, and retention of recovery copies.

## Configure a private CA for an optional Ubuntu Pro service

This section applies only if your deployment uses Ubuntu Pro resource-token validation against a service with a private CA. It is not required for the local auth-token and machine-token workflow.

For either charm, provide the issuing CA using `contracts.ca`. The value must be base64-encoded PEM certificate content:

```shell
juju config livepatch contracts.ca="$(base64 -w0 ca-cert.pem)"
```

In a Juju bundle, use `contracts.ca: include-base64://path/to/ca-cert.pem` instead. Keep the service URL consistent with the certificate's hostname. This trust configuration is separate from the CA that clients and administrators use to verify the Livepatch TLS frontend.

## Use the latest version

Keep the deployed charm or standalone snap on the current `latest/stable` revision. Follow {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>` for manual updates, standalone snap automatic-update controls, and the risks of delaying updates.

Use Juju to update charm-managed deployments. Do not manually refresh or unhold the server snap managed by the machine charm.

## Secure admin credentials

Use individual administrator accounts and strong passwords. Generate a bcrypt username-and-hash pair with `htpasswd` from the `apache2-utils` package. Enter the password interactively rather than putting it in the command arguments:

```shell
htpasswd -nBC 10 admin
```

Replace `admin` with the intended username. Treat the resulting hash as sensitive: it can be used for offline password guessing.

### Machine and Kubernetes charms

Store the username-and-hash list in a Juju secret. Create a file named `admin-users.secret.yaml` outside version control, with permissions `0600` before adding the following content:

```yaml
value: 'admin:<bcrypt-hash>'
```

Replace the example with the complete output from `htpasswd`. For multiple users, put comma-separated `username:bcrypt-hash` pairs in the same `value` string.

Create the secret and grant the Livepatch application access:

```shell
juju add-secret livepatch-admin-users --file admin-users.secret.yaml
juju grant-secret livepatch-admin-users livepatch
```

Use the secret URI returned by `juju add-secret`:

```shell
juju config livepatch auth.basic.enabled=true auth.basic.users-secret='<secret-uri>'
```

The secret-backed option takes precedence over `auth.basic.users`. After verifying administrator access, reset any obsolete plain-configuration value:

```shell
juju config livepatch --reset auth.basic.users
```

Resetting the option does not remove historical configuration exports or backups. Remove the temporary secret file when finished and retain any required copy only in an approved credential store. Restrict who can read or change Juju secrets and application configuration.

### Standalone snap

Configure `auth.basic.enabled` and `auth.basic.users` through the supported snap settings, with the `lp.` prefix. Do not edit generated workload configuration files. Restrict privileged access to the host, snap configuration, and snap data, because these can contain credentials.

See the [administration tool setup guide](/server/how-to-guides/security/setup-administration-tool.md#password-authentication) for password-authentication configuration and login.

## Configure request limits

The server's request-admission controls limit simultaneous work:

* `server.concurrency-limit`: Maximum number of requests being processed at once.
* `server.burst-limit`: Maximum total number of requests being processed or waiting to be processed.

The packaged configuration defaults differ:

| Deployment | `server.concurrency-limit` | `server.burst-limit` |
| ---------- | -------------------------- | -------------------- |
| Machine and Kubernetes charms | 1000 | 500 |
| Standalone snap | 200 | 500 |

For example, a processing limit of 200 and a total limit of 500 allow up to 300 waiting requests. To configure that example for either charm:

```shell
juju config livepatch server.concurrency-limit=200 server.burst-limit=500
```

For the standalone snap:

```shell
sudo snap set canonical-livepatch-server lp.server.concurrency-limit=200 lp.server.burst-limit=500
```

Tune the values for your workload, memory, CPU, and database capacity; these example values are not a sizing recommendation. Monitor latency, resource use, and HTTP 503 responses when admission limits reject excess work. Use [centralized logging and monitoring](/server/how-to-guides/operations/configure-logging-and-monitoring.md) to detect sustained overload.

These are concurrency limits, not requests-per-second limits or complete denial-of-service protection. Configure appropriate connection limits, request-rate limits, and timeouts at your ingress or reverse proxy, and bound client retries so that overload does not generate an increasing retry load.

## Restrict network access

Apply firewall rules, proxy access controls, or enforced Kubernetes network policies to your deployment:

* Allow administrator endpoints only from trusted administration networks. Preserve the client-facing routes needed for registration, reporting, and patch downloads.
* Allow `/metrics` access only from authorized monitoring infrastructure.
* Keep database ports and storage-management endpoints off public networks.
* Restrict the HTTP backend to the frontend and other explicitly authorized callers.
* Limit outbound access to configured dependencies. Connected deployments need their configured upstream for synchronization; air-gapped deployments use offline patch imports and local dependencies.

## Maintain strict privileged access

### Juju-managed deployments

Restrict controller, model, configuration, secret, and action access to authorized operators. Relation data and secrets can contain credentials; access to Juju is part of the deployment's security boundary.

For the machine charm, restrict SSH, `sudo`, and Juju execution access to the managed machines. Manage the workload through the charm rather than making persistent manual changes to its snap or generated configuration.

For the Kubernetes charm, use Kubernetes RBAC to restrict access to the namespace, secrets, and pod execution. Apply `NetworkPolicy` resources with a network plugin that enforces them, and verify that required ingress, database, storage, and monitoring traffic still works.

### Standalone snap

The snap uses `strict` confinement. Confinement does not replace host access controls: restrict SSH and `sudo` access, and protect server data under `$SNAP_COMMON` and `$SNAP_DATA`, including backups and snapshots.

### Client machines

Protect client authentication material and restrict privileged access to client machines. Anyone who obtains a valid token may be able to authenticate as that client. For registration instructions, see the [client setup guide](/server/how-to-guides/operations/use-livepatch-client-with-on-prem-server.md).

Aggregate operational logs centrally and restrict access to the log destination to help detect unexpected configuration changes, authentication failures, and suspicious administrative activity. See [logging and monitoring](/server/how-to-guides/operations/configure-logging-and-monitoring.md).

## Reset to original state

Use the secure configuration guide to choose between a {ref}`configuration-only reset <server-how-to-guides-reset-livepatch-server-configuration>`, which preserves stored data, and a {ref}`full reset <server-how-to-guides-fully-reset-livepatch-server>`, which removes application data before a fresh deployment.

Removing and redeploying the application alone does not clear an external database or patch store.
