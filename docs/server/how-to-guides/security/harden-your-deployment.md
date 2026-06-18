---
myst:
  html_meta:
    description: "How to harden your deployment with Livepatch on-prem."
---


(server-how-to-guides-harden-your-livepatch-server-deployment)=

# Harden a Livepatch Server deployment

The Livepatch Server is available as a Kubernetes charm and a strict-confinement snap for on-premises deployment. The following sections provide guidance on securing the server in production deployments.

## Enable TLS termination

The Livepatch Server does not natively terminate TLS. In on-premises deployments where clients and the server are on a trusted internal network, TLS between them is optional. However, if the server is accessible from untrusted networks or admin API endpoints are exposed, a TLS-terminating reverse proxy, Kubernetes ingress controller, or load balancer should be placed in front of the server.

For **charm deployments**, TLS termination is configured through an ingress integration. The charm supports two ingress interfaces:

* **`nginx-route`** (legacy): Integration with the [Nginx Ingress Integrator charm](https://charmhub.io/nginx-ingress-integrator). Set `ingress-interface=legacy-nginx-route` (default).
* **`ingress`** (modern): Integration with the [Traefik K8s charm](https://charmhub.io/traefik-k8s) or [Gateway API Integrator charm](https://charmhub.io/gateway-api-integrator). Set `ingress-interface=ingress`.

Example using the modern ingress integration:

```shell
juju config canonical-livepatch-server-k8s ingress-interface=ingress
juju integrate canonical-livepatch-server-k8s traefik-k8s
```

For **snap deployments**, a reverse proxy (for example, Nginx or HAProxy) can be configured with TLS certificates to terminate HTTPS before forwarding traffic to the server.

Without TLS termination, all communication -- including authentication credentials -- is transmitted in plaintext. This is acceptable on a fully trusted internal network, but not recommended when any network segment is untrusted.

For step-by-step TLS setup instructions, see the [TLS configuration guide](/server/how-to-guides/security/setup-tls.md).

### Federation (chained servers)

In [federated deployments](/server/how-to-guides/patch-management/chain-livepatch-servers.md) where multiple on-premises servers are chained, TLS should also be enabled between them to protect the exchange of tier and patch data. Configure TLS termination on the upstream server just as for client-facing traffic.

## Encrypt database connections

TLS for PostgreSQL connections is supported through the connection string. Include `sslmode=verify-full` (or at minimum `sslmode=require`) in the database connection string to encrypt traffic between the server and PostgreSQL.

For charm deployments, database credentials are obtained through Juju relations (`database` interface with the [PostgreSQL K8s Charm](https://charmhub.io/postgresql-k8s)). TLS for the database connection is managed by the PostgreSQL charm's [TLS integration](https://canonical-charmed-postgresql-k8s.readthedocs-hosted.com/14/tutorial/index.html#enable-encryption-with-tls). The Livepatch charm does not expose separate database credential configuration -- all credentials are exchanged through the Juju relation protocol.

## Encrypt patch storage connections

* **S3**: Set `patch-storage.s3-secure` to `true` to enforce HTTPS for all S3 communication.
* **Swift**: Configure `patch-storage.swift-auth-url` to use an HTTPS endpoint.
* **Filesystem/PostgreSQL**: No additional configuration needed as storage is local to the server or database.

## Configure a custom CA certificate for the contracts service

When the Ubuntu Pro service uses a certificate issued by a private CA (common in air-gapped or enterprise environments), provide the CA certificate via the `contracts.ca` configuration option.

The certificate must be base64-encoded (base64 of the PEM-encoded certificate, from `-----BEGIN` to `-----END`). In a Juju bundle, use `include-base64://` to include the certificate file:

```
applications:
  canonical-livepatch-server-k8s:
    options:
      contracts.ca: include-base64://path/to/ca-cert.pem
```

Or via the Juju CLI:

```shell
juju config canonical-livepatch-server-k8s contracts.ca="$(base64 -w0 ca-cert.pem)"
```

The charm decodes the certificate and installs it into the container's system CA trust store at `/usr/local/share/ca-certificates/`, then runs `update-ca-certificates --fresh` to apply it.

## Use the latest version

### Charm

Ensure the charm is deployed from the latest stable revision:

```shell
juju refresh canonical-livepatch-server-k8s --channel=latest/stable
```

### Snap

The snap is installed from the `latest/stable` channel. To verify the installed version matches the latest stable release:

```shell
snap info canonical-livepatch-server | grep -E 'installed|latest/stable'
```

If the versions do not match, update the snap:

```shell
sudo snap refresh canonical-livepatch-server --channel=latest/stable
```

By default, the snapd daemon checks for updates four times a day and applies them automatically. Modifying this behavior could leave the system vulnerable to unpatched security issues.

## Secure admin credentials

Admin users are configured with bcrypt-hashed passwords in the server configuration. To generate a bcrypt hash for a new admin password:

```shell
htpasswd -nbBC 10 "" "your-password" | cut -d: -f2
```

For charm deployments, configure admin credentials through the Juju configuration:

```shell
juju config canonical-livepatch-server-k8s auth.basic.enabled=true
juju config canonical-livepatch-server-k8s auth.basic.users="admin:\$2y\$10\$hashedpassword"
```

The `auth.basic.users` option accepts a comma-separated list of `user:bcrypt-hash` pairs. Juju secrets should be used where possible to avoid exposing credentials in Juju configuration output.

For snap deployments, ensure that the configuration file (typically at `$SNAP_COMMON/config.yaml`) has restrictive file permissions (readable only by the snap's runtime user).

## Configure request limits

The HTTP governor protects the server from overload. Configure appropriate limits for the deployment:

* `server.concurrency-limit` (default: 1000): Maximum number of requests processed simultaneously.
* `server.burst-limit` (default: 500): Maximum number of concurrently incoming requests. Requests exceeding this are queued up to `concurrency-limit - burst-limit` (default: 500), and rejected beyond that.

For charm deployments:

```shell
juju config canonical-livepatch-server-k8s server.concurrency-limit=1000
juju config canonical-livepatch-server-k8s server.burst-limit=500
```

These values should be tuned based on the expected load and available resources.

## Restrict network access

In production deployments:

* Admin API endpoints should only be accessible from trusted networks.
* The `/metrics` endpoint (Prometheus) should be restricted to monitoring infrastructure.
* Database ports should not be exposed to the public network.

## Maintain strict privileged access

### Snap

The snap uses `strict` confinement, restricting the server to only the `network` and `network-bind` interfaces.

All server data is stored under the `$SNAP_COMMON` and `$SNAP_DATA` directories. Maintain strict privileged access controls on the host machine to prevent unauthorized access to this data.

### Client machines

When a Livepatch Client is registered with the on-premises server, an authentication token is stored on the client machine. Ensure only authorized users have `sudo` access to hosts running the Livepatch Client, as the token grants the ability to authenticate against the server. For details on client registration, see the [client setup guide](/server/how-to-guides/operations/use-livepatch-client-with-on-prem-server.md).

### Charm (Kubernetes)

The charm deploys the server as a container. The server listens on port 8080 internally. Apply Kubernetes security best practices:

* Use `NetworkPolicy` resources to restrict pod-to-pod communication.
* Use Kubernetes RBAC to limit access to the server's namespace and secrets.

The charm stores its internal state (including the database connection string) in the Juju peer relation data. Access to the Juju controller and model should be restricted to authorized operators.

## Reset to original state

### Snap

To reset the snap configuration to its default state:

```shell
sudo snap set canonical-livepatch-server key=
```

Alternatively, remove and reinstall the snap. See [Decommission](/server/how-to-guides/operations/decommission.md) for details.

### Charm

To reset the charm to its default configuration:

```shell
juju config canonical-livepatch-server-k8s --reset <key>
```

To fully reset, remove the application and redeploy.