---
myst:
  html_meta:
    description: "Explore the security architecture of the Livepatch on-premises server including trust boundaries, authentication, cryptographic mechanisms, and security recommendations."
---

(server-explanation-security-overview)=

# Security overview

> See also: {ref}`Authentication and authorization <server-reference-livepatch-server-authentication>`, {ref}`Security lifecycle <server-explanation-security-lifecycle>`, {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`

The Livepatch on-premises server synchronizes kernel patches from Canonical's hosted Livepatch service, or from another on-premises server, and serves them to Livepatch Clients within the operator's infrastructure. It is available as a Kubernetes charm, a machine charm, and a standalone snap.

This document describes the server from a security perspective, outlines its built-in protections and risks, and links to more detailed guidance.

## System overview

The on-premises Livepatch Server is the central component of a deployment, coordinating between administrators, clients, its upstream patch source, and supporting services.

![Security architecture showing clients, administrator access, the server deployment, backend services, management access, and optional direct patch transfers across trust boundaries.](/_static/images/server-security-architecture.svg)

The diagram shows logical security contexts, not a required network layout. Dashed boxes mark trust boundaries, solid arrows show service communication or management access, and dashed arrows show optional direct patch transfers.

### Components

- **Ingress or reverse proxy**: Forwards client and administrator requests to the server. It can terminate TLS and enforce network restrictions and request limits.
- **On-premises Livepatch Server**: Manages patches and serves them to clients. It exposes HTTP APIs for the Livepatch Client, the Livepatch Admin Tool, and server-to-server synchronization.
- **PostgreSQL database**: Stores authentication tokens, patch metadata, machine registrations, machine reports, and macaroon root keys.
- **Patch storage backend**: Stores patch files. See {ref}`Patch storage <server-reference-patch-storage>` for the supported backends.
- **Livepatch Admin Tool**: CLI used by administrators to manage tokens, tiers, patches, and webhooks.
- **Livepatch Client**: Queries the server for patches and applies them to the running kernel.
- **Upstream patch source**: Canonical's hosted Livepatch service, or another on-premises server, from which the server synchronizes patches.
- **Ubuntu Pro service**: Optional. When the contracts service is enabled, the server validates client resource tokens against Ubuntu Pro or a Pro Airgapped Server.
- **Management infrastructure**: Juju for charm deployments, or host administrators for the standalone snap.
- **Observability services**: Optional Prometheus, Grafana, and Loki integrations.

Patch transfers do not always pass through the server. When configured to redirect downloads, clients download patches directly from the URL in `server.url-template`. With cloud storage backends, the admin tool uploads patches directly using short-lived credentials issued by the server. The storage endpoint is therefore a separate security boundary.

### Trust boundaries

TLS is optional within a trusted internal network, and recommended across untrusted networks.

| Boundary | Security properties |
| :---- | :---- |
| Client to server | Clients register with an auth token issued by the server, or with a resource token when the contracts service is enabled, then authenticate with a machine token. Tier and affordance checks restrict access to compatible patches. |
| Administrator to server | Administrators use Basic Auth to obtain macaroons. Restrict admin API access to trusted networks. |
| Proxy to server | The server does not terminate TLS, so the proxy forwards plain HTTP. Keep this hop on a trusted network and block direct access to the backend listener. |
| Server to database and patch storage | TLS is supported but optional. Filesystem storage relies on host access controls. |
| Server to upstream | Authenticated with a resource token (hosted service) or a sync token (on-premises upstream) configured through `patch-sync.token` or `patch-sync.token-secret`. Proxy support is available through `patch-sync.proxy.*`. |
| Management to workload | Juju and host administrators can change the workload and its credentials. Application authentication does not restrict them. |
| Server to observability services | Metrics and logs leave the workload when monitoring integrations are enabled. |

For the Kubernetes charm, TLS termination is provided through the legacy `nginx-route` integration, or the `ingress` integration with Traefik or the Gateway API Integrator. The machine charm uses HAProxy.

In {ref}`federated deployments <server-how-to-guides-chain-on-prem-servers>`, downstream servers authenticate to an upstream on-premises server with sync tokens that it issues. In air-gapped deployments, patches enter through an [offline transfer workflow](/server/how-to-guides/patch-management/distribute-patches-offline.md).

## Built-in protections

- **Multi-layered authentication**: Macaroons (Basic Auth-backed) for administrators; auth, machine, and resource tokens for clients; resource tokens or sync tokens for upstream synchronization. See the {ref}`authentication reference <server-reference-livepatch-server-authentication>`.
- **Request limiting**: An HTTP governor limits concurrent requests and can briefly queue excess requests. See [Configure request limits](/server/how-to-guides/security/harden-your-deployment.md#configure-request-limits).
- **Input validation**: API inputs are validated using schema, format, length, and type checks.
- **Structured security event logging**: Authentication, authorization, token lifecycle, admin, and system events are logged as structured JSON. See the {ref}`logging schema <server-reference-logging-schema>`.
- **Prometheus metrics**: The `/metrics` endpoint exposes authentication, latency, database error, queue, and overload metrics.
- **(charms) Log redaction**: Both charms redact credentials, authorization headers, and secret values from their operator logs. Server application logs are not redacted.
- **(Kubernetes charm) Observability integrations**: The `metrics-endpoint`, `grafana-dashboard`, and `logging` integrations provide Prometheus scraping, a Grafana dashboard, and Loki log forwarding.
- **(Kubernetes charm) Health checks**: A Pebble HTTP check probes `/debug/info` every 60 seconds to detect and recover from service failures.
- **(snap) Strict confinement**: The standalone snap and the snap managed by the machine charm use strict confinement. The server daemon is restricted to the `network` and `network-bind` interfaces.

TLS termination, backend TLS, firewall rules, and management access controls require operator configuration. See the {ref}`hardening guide <server-how-to-guides-harden-your-livepatch-server-deployment>`.

## Risks

- **TLS not natively terminated**: The server relies on a reverse proxy, ingress controller, or load balancer for TLS. If any network segment is untrusted or admin APIs are exposed beyond the internal network, configure TLS termination to protect credentials in transit. See {ref}`Set up TLS <server-how-to-guides-how-to-setup-tls-for-livepatch-on-prem>`.
- **TLS optional for database and storage connections**: TLS is supported but not enforced for PostgreSQL and remote storage backends. Enable it wherever these connections cross untrusted networks.
- **No native account lockout**: The server does not track failed logins or lock accounts. Implement brute-force protection at the network layer, for example with firewall rules and ingress rate limiting.
- **Configuration-based secret management**: Admin, storage, and upstream credentials are supplied through configuration, environment variables, or Juju secrets. Use Juju secrets where supported and restrict access to configuration and the workload environment. Charm log redaction does not protect values visible through workload inspection.
- **Privileged access**: Access to admin APIs, configuration, or management infrastructure allows changing patch availability, issuing tokens, and reading stored data. Restrict this access to authorized operators.
- **Resource exhaustion**: The request governor limits application concurrency, but does not protect a saturated network or backend services. Configure limits at the proxy as well, and monitor overload metrics.
- **Stored and exported data**: Machine reports, logs, backups, and configuration exports can disclose credentials or fleet information. Restrict access to them and apply your retention policy.
- **Missing audit records**: Without retained security logs, unauthorized changes can go unnoticed. Use individual administrator accounts and forward logs to a central aggregator. See {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.

## Security of data

### Data collection and storage

The Livepatch Server stores the following data:

- **Client registration**: Auth tokens, machine tokens, machine IDs, and tier assignments.
- **Sync tokens**: Tokens issued to downstream servers.
- **Patch metadata**: Patch versions, checksums, distribution and architecture mappings, and storage references.
- **Machine reports**: When database reporting is enabled, client machine IDs, patch state, and system information.
- **Admin credentials**: Bcrypt-hashed passwords for Basic Auth users, from deployment configuration or Juju secrets.
- **Macaroon root keys**: Used for admin sessions, with a 24-hour expiry policy.

Except for admin credentials, this data is stored in PostgreSQL. Upstream and storage credentials are held in the deployment configuration. Patch files are stored in the configured {ref}`patch storage backend <server-reference-patch-storage>`.

During synchronization, the server sends its deployment ID, machine counts, and patch-selection information to its upstream. Detailed machine reports are forwarded only when `patch-sync.send-machine-reports` is enabled. See {ref}`Data sent to Canonical servers <server-reference-data-sent-to-canonical-servers>`.

### Data retention and deletion

Machine-report storage is disabled by default. When enabled, reports are retained for `machine-reports.database.retention-days`: 10 days by default for the charms, and 7 days for the standalone snap. Cleanup runs periodically on the leader, in batches, so reports can outlive the retention period if cleanup falls behind. Setting the retention period to `0` disables cleanup, and disabling report storage does not delete existing reports.

Report cleanup does not remove tokens, registrations, patches, logs, backups, or exported copies. Manage these under your own retention policy. See {ref}`Decommission the server securely <server-how-to-guides-decommission-livepatch-server-securely>` for removing a deployment and its data.

### Cryptographic mechanisms

The following cryptographic technologies are used by the Livepatch Server and admin tool:

**Authentication:**

| Mechanism | Algorithm | Go packages |
| :---- | :---- | :---- |
| Basic Auth password verification | bcrypt | `golang.org/x/crypto/bcrypt`, `crypto/subtle` |
| Macaroons (HMAC signatures) | HMAC-SHA256, XSalsa20-Poly1305 | `crypto/hmac`, `crypto/sha256`, `golang.org/x/crypto/nacl/secretbox` |
| Macaroon Bakery (public key crypto) | Ed25519, XSalsa20-Poly1305 | `golang.org/x/crypto/nacl/box`, `golang.org/x/crypto/curve25519` |
| Token generation (UUIDv4) | Random number generation | `github.com/google/uuid` |

**Data protection:**

| Mechanism | Algorithm | Go packages |
| :---- | :---- | :---- |
| Machine ID hashing in forwarded machine reports | HMAC-SHA256 | `crypto/hmac`, `crypto/sha256` |
| S3 upload credentials (presigned POST policies) | AWS Signature Version 4 (HMAC-SHA256) | `crypto/hmac`, `crypto/sha256` |
| Swift upload credentials (TempURLs) | HMAC-SHA1, the Swift TempURL default | `crypto/hmac`, `crypto/sha1` |
| Server outbound TLS, and admin tool TLS without a custom CA | TLS 1.2 or later (Go default) | `crypto/tls`, `crypto/x509` |
| Admin tool TLS with a custom CA (`--ca` or `LIVEPATCH_CA_CRT`) | TLS 1.3 or later | `crypto/tls`, `crypto/x509` |

**External packages providing cryptographic functionality:**

- [`gopkg.in/macaroon.v2`](https://gopkg.in/macaroon.v2): Macaroon creation and verification
- [`github.com/go-macaroon-bakery/macaroon-bakery`](https://github.com/go-macaroon-bakery/macaroon-bakery): Higher-level macaroon bakery with public key cryptography
- [`golang.org/x/crypto`](https://golang.org/x/crypto): bcrypt, NaCl box/secretbox, curve25519
- Go standard library `crypto/*`: TLS, HMAC, SHA-1, SHA-256, X.509

### Data at rest

Admin passwords are bcrypt-hashed, but the server does not encrypt its database, configuration, or patch storage. Use encryption at rest provided by your hosts, database, and storage services, and protect backups and exports in the same way. See {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`.

### Data in transit

The server serves HTTP only. TLS for clients and the admin tool is terminated by the frontend.

| Communication path | TLS |
| :---- | :---- |
| Livepatch Client to frontend | Optional (operator-configured on the frontend) |
| Livepatch Admin Tool to frontend | Optional (operator-configured on the frontend) |
| Frontend to Livepatch Server | Not supported; keep on a trusted network |
| Livepatch Server to upstream | Optional (HTTPS upstream URL) |
| Livepatch Server to Ubuntu Pro service | Optional (HTTPS `contracts.url`) |
| Livepatch Server to PostgreSQL | Optional (connection string for the snap; PostgreSQL charm TLS for the charms) |
| Livepatch Server to S3 | Optional (`patch-storage.s3-secure`) |
| Livepatch Server to Swift, Google Cloud Storage, or Azure Blob Storage | Optional (HTTPS endpoints) |

## Security support

The Livepatch Server is released as a Kubernetes charm, a machine charm, and a standalone snap. Keep deployments on the current revision in `latest/stable`. See {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>`.

Releases are made ad-hoc as features and bug fixes are implemented. When a security vulnerability in an upstream dependency is detected, a best-effort attempt is made to upgrade the dependency to the latest version that fixes the vulnerability. This security fix is then included in the next ad-hoc release.

A fix for a reported security issue that is directly present in the on-premises Livepatch Server can be prioritized and released as a minor version upgrade for the charms and snap.

Canonical customers can obtain support through the [Canonical support portal](https://portal.support.canonical.com/). To report a potential vulnerability, follow {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`.

## Related security documentation

- {ref}`Authentication reference <server-reference-livepatch-server-authentication>`: Credential types and authentication flows.
- {ref}`Configure the server securely <server-how-to-guides-configure-livepatch-server-securely>`: Initial configuration and reset procedures.
- {ref}`Set up administrator authentication <server-how-to-guides-how-to-setup-livepatch-on-prem-server-administration-tool>`: Administrator accounts and credential changes.
- {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`: Securing production deployments.
- {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>`: Keeping deployments up to date.
- {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`: Storage protection, backup, and restore.
- {ref}`Integrate with other services securely <server-how-to-guides-integrate-services-securely>`: Securing connections to supporting services.
- {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`: Security event logging and Prometheus monitoring.
- {ref}`Logging schema <server-reference-logging-schema>`: Structured log fields and security-event identifiers.
- {ref}`Decommission the server securely <server-how-to-guides-decommission-livepatch-server-securely>`: Removing the server and its data.
- {ref}`Security lifecycle <server-explanation-security-lifecycle>`: How security updates are released and delivered.
- {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`: Reporting security vulnerabilities.
