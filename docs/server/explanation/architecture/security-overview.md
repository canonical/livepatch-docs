---
myst:
  html_meta:
    description: "Explore the security architecture of the Livepatch on-premises server including trust boundaries, authentication, cryptographic mechanisms, and security recommendations."
---

(server-explanation-security-overview)=

# Security overview

> See also: {ref}`Authentication and authorization <server-reference-livepatch-server-authentication>`, {ref}`Security lifecycle <server-explanation-security-lifecycle>`, {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`

The Livepatch on-premises server synchronizes kernel patches from Canonical's hosted Livepatch service, or from another on-premises server, and serves them to Livepatch Clients within the operator's infrastructure. It is available as a Kubernetes charm, a machine charm, and a standalone snap.

This document provides an overview of the security features and practices implemented in the on-premises Livepatch Server. It describes the system from a security perspective, outlines built-in protections, and references more detailed documentation for specific cryptographic approaches and operational guidance.

## System overview

The on-premises Livepatch Server acts as the central component in an on-premises Livepatch deployment, coordinating between administrators, clients, Canonical's hosted service, and external services.

![Security architecture showing clients, administrator access, the server deployment, backend services, management access, and optional direct patch transfers across trust boundaries.](/_static/images/server-security-architecture.svg)

The diagram shows logical security contexts, not a required network layout. Dashed boxes mark trust boundaries, solid arrows show service communication or management access, and dashed arrows show optional direct patch transfers.

### Components

The following components are part of a Livepatch on-premises deployment:

- **Ingress or reverse proxy**: Forwards client and administrator requests to the server. It can terminate TLS and enforce network restrictions and request limits.
- **On-premises Livepatch Server**: The core service that manages patches and serves them to clients within the operator's infrastructure. It exposes HTTP APIs for the Livepatch Client, the Livepatch Admin Tool, and server-to-server communication.
- **PostgreSQL database**: Stores authentication tokens, patch metadata, machine registrations, machine reports, and macaroon root keys.
- **Patch storage backend**: Stores patch files. See {ref}`Patch storage <server-reference-patch-storage>` for the supported backends.
- **Livepatch Admin Tool**: CLI tool used by administrators to manage the server (tokens, tiers, patches, webhooks).
- **Livepatch Client**: The endpoint agent that queries the server for patches and applies them to the running kernel.
- **Canonical hosted Livepatch service**: Canonical's hosted Livepatch service from which the on-premises server synchronizes patch data.
- **Ubuntu Pro service**: External service used to validate Ubuntu Pro resource tokens from clients.
- **Management and observability**: Juju or host administrators manage the workload. Optional Prometheus, Grafana, and Loki integrations collect metrics and logs.

When configured to redirect downloads, clients download patches directly from the URL in `server.url-template`. With cloud storage backends, the admin tool uploads patches directly using short-lived credentials issued by the server. The storage endpoint is therefore a separate security boundary.

### Trust boundaries

The following trust boundaries are relevant to the security posture of the system:

- **Client network to Livepatch Server**: In on-premises deployments, Livepatch Clients and the server typically reside on the same internal network. TLS between clients and the server is optional, and is terminated by a frontend: the Kubernetes charm uses the `nginx-route` integration (legacy) or the `ingress` integration (Traefik or the Gateway API Integrator), and the machine charm uses HAProxy.
- **Frontend to Livepatch Server**: The server does not terminate TLS, so the frontend forwards plain HTTP. Keep this hop on a trusted network and block direct access to the backend listener.
- **Admin network to Livepatch Server**: Admin API requests should originate from trusted networks. Authentication is enforced through macaroons obtained via Basic Auth.
- **Livepatch Server to PostgreSQL**: Database connections cross a trust boundary. TLS is supported but optional for this connection.
- **Livepatch Server to external services**: The charms support a custom CA certificate (`contracts.ca`) for verifying the Ubuntu Pro service certificate. Connections to remote patch storage backends support TLS.
- **Livepatch Server to Pro Airgapped Server**: For air-gapped environments, the charm can integrate with a Pro Airgapped Server via the `pro-airgapped-server` Juju relation to validate resource tokens without external network access.
- **Patch synchronization with Canonical's hosted service**: The on-premises server synchronizes patches from Canonical's hosted Livepatch service using either a contract resource token or a sync token for authentication. Resource tokens are obtained from the Ubuntu Pro contracts service using the `get-resource-token` Juju action (charm) or by setting the contract token via `snap set canonical-livepatch-server token=<CONTRACT_TOKEN>` (snap), which triggers the configure hook to exchange it for a resource token automatically. Sync tokens are issued by the upstream server's admin tool. The active token is configured via `patch-sync.token`. Proxy support is available for environments where direct connectivity is restricted (`patch-sync.proxy.*` configuration).
- **Federation (hierarchical on-premises deployments)**: In federated deployments, a designated on-premises server pulls patches from Canonical's hosted service. Downstream on-premises servers (for example, in different data centers) pull patches from this designated server. To authorize these connections, the administrator uses the admin tool to issue sync tokens from the upstream server to each downstream server.
- **Management to workload**: Juju and host administrators can change the workload, its configuration, and its credentials. Application authentication does not restrict them.
- **Livepatch Server to observability services**: Metrics and logs leave the workload when monitoring integrations are enabled.

## Built-in protections

The Livepatch Server incorporates several mechanisms to ensure the security of its operations:

- **Multi-layered authentication**: Prevents unauthorized administration, client access, and patch synchronization. Administrators use macaroons (Basic Auth-backed); clients use auth, machine, or resource tokens, with tier and affordance checks; upstream synchronization uses resource tokens or sync tokens. See the {ref}`Authentication reference <server-reference-livepatch-server-authentication>`.
- **Request limiting**: An HTTP governor limits concurrent request processing and queues excess requests up to a configurable burst limit, protecting against overload and resource exhaustion.
- **Input validation**: All API inputs are validated using strict schemas with regex patterns, length limits, and type checks across validation modules. This mitigates injection attacks and malformed request handling.
- **Structured security event logging**: Authentication successes and failures, authorization decisions, token lifecycle events, admin activity, and system events are logged as structured JSON, so that unauthorized access and changes can be detected and investigated. See the {ref}`logging schema <server-reference-logging-schema>`.
- **Prometheus monitoring**: The `/metrics` endpoint exposes authentication, latency, database error, queue, and overload metrics, so that attacks, overload, and backend failures can be detected and alerted on.
- **(charms) Log redaction**: Both charms redact credentials, authorization headers, and secret values from their operator logs, reducing credential disclosure through logs. Server application logs are not redacted.
- **(Kubernetes charm) Observability integrations**: The charm can send metrics, logs, and dashboards to the Canonical Observability Stack (COS), so that evidence survives the loss of a unit.
- **(Kubernetes charm) Container health checks**: The charm configures a Pebble HTTP health check against the `/debug/info` endpoint, running every 60 seconds, to detect and recover from service failures.
- **(snap) Strict snap confinement**: The snap package uses `strict` confinement, restricting the server to only the `network` and `network-bind` interfaces. This limits the impact of a potential compromise by preventing access to the host system beyond network operations.
- **Patch integrity**: Clients verify each patch's SHA-256 checksum and Canonical's signature on the kernel module before applying it, so tampered patch files are rejected. See {ref}`Patch security <client-reference-patch-security>`.

TLS termination, backend TLS, firewall rules, and management access controls require operator configuration. See the {ref}`hardening guide <server-how-to-guides-harden-your-livepatch-server-deployment>`.

## Risks

While the Livepatch Server is designed with security in mind, the following risks should be considered:

- **TLS not natively terminated**: The Livepatch Server does not terminate TLS itself. When TLS is required, it relies on an external reverse proxy, Kubernetes ingress controller, or load balancer for TLS termination. In on-premises deployments where clients and the server are on a trusted internal network, operators may choose to run without TLS. However, if the network is not fully trusted or admin API endpoints are exposed beyond the internal network, configuring TLS termination is strongly recommended to protect authentication credentials in transit.
- **TLS optional for database and storage connections**: Connections between the server and PostgreSQL, S3, and Swift storage backends support TLS but do not enforce it. Running without TLS on these connections in environments with untrusted network segments increases the risk of credential or data interception. Enabling TLS for all backend connections is recommended.
- **Disabled or weakened defaults**: Setting `server.concurrency-limit` to `0` disables request limiting, raising `server.log-level` above `info` drops security events, setting `patch-storage.s3-secure` to `false` sends S3 traffic unencrypted, and `--skip-tls-verify` disables certificate checks in the admin tool. Review the {ref}`configuration reference <server-reference-configuration>` before changing these.
- **No native account lockout**: The server does not implement failed login attempt tracking or temporary lockouts. Admin users are defined in configuration files with bcrypt-hashed passwords. Brute-force protection should be implemented at the network layer (firewall rules, ingress rate limiting).
- **Configuration-based secret management**: Admin credentials and storage credentials are provided through configuration options or environment variables. All secrets are passed to the workload container as environment variables. Operators should use Juju secrets or Kubernetes secrets to protect sensitive configuration options. The charm's log redaction module mitigates the risk of secrets appearing in charm logs, but secrets may still be visible in Pebble plan output or container environment inspection.
- **Insufficient access control**: Access to admin APIs, configuration, or management infrastructure allows changing patch availability, issuing tokens, and reading stored data. Use individual administrator accounts and restrict this access, as described in {ref}`administrator setup <server-how-to-guides-how-to-setup-livepatch-on-prem-server-administration-tool>`, {ref}`client registration <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`, and the {ref}`hardening guide <server-how-to-guides-harden-your-livepatch-server-deployment>`.
- **Untrusted input in integrations**: The server validates API input, but scripts and automation that call the admin tool or API must validate their own input. Building commands or requests from untrusted values can cause injection or unintended changes.
- **Denial of service**: The request governor limits application concurrency, but does not protect a saturated network or backend services. Configure connection and rate limits at the frontend, and monitor overload metrics. See {ref}`Integrate with other services securely <server-how-to-guides-integrate-services-securely>`.
- **Incorrect data handling**: Machine reports, logs, backups, and configuration exports can disclose credentials or fleet information. Restrict and encrypt them as described in the {ref}`hardening guide <server-how-to-guides-harden-your-livepatch-server-deployment>` and {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`.
- **Tampering**: Patch files are protected by client-side verification, but configuration, tokens, tier assignments, and database records are not. Anyone with write access to them can change which clients receive patches. Restrict write access, and alert on unexpected administrative events using {ref}`logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.
- **Repudiation**: Without retained security logs, or with shared administrator accounts, actions cannot be attributed to individuals. Use individual accounts and forward logs to a central aggregator with restricted access.

## Security of data

### Data collection and storage

The Livepatch Server collects and stores the following categories of data:

- **Machine registration data**: Machine IDs, authentication tokens, and tier information for registered Livepatch Clients.
- **Sync tokens**: Tokens issued to downstream servers.
- **Patch metadata**: Patch versions, checksums, distribution/architecture mappings, and storage references.
- **Machine reports**: When database reporting is enabled, client machine IDs, patch state, and system information.
- **Admin credentials**: Bcrypt-hashed passwords for Basic Auth admin users.
- **Macaroon root keys**: Stored in PostgreSQL with a 24-hour expiry policy, used for Basic Auth-backed admin session authentication.

Except for admin credentials, which are held in the deployment configuration or Juju secrets, this data is stored in PostgreSQL. Patch files are stored in the configured {ref}`patch storage backend <server-reference-patch-storage>`. During synchronization, the server sends its deployment ID, machine counts, and patch-selection information to its upstream; see {ref}`Data sent to Canonical servers <server-reference-data-sent-to-canonical-servers>`.

### Data retention and deletion

Machine-report storage is disabled by default. When enabled, reports are retained for `machine-reports.database.retention-days`: 10 days by default for the charms, and 7 days for the standalone snap. Cleanup runs periodically on the leader, in batches, so reports can outlive the retention period if cleanup falls behind. Setting the retention period to `0` disables cleanup, and disabling report storage does not delete existing reports.

Report cleanup does not remove tokens, registrations, patches, logs, backups, or exported copies. Manage these under your own retention policy. See {ref}`Decommission the server securely <server-how-to-guides-decommission-livepatch-server-securely>` for removing a deployment and its data.

### Cryptographic mechanisms

The following cryptographic technologies are used by the Livepatch Server:

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
| Machine ID pseudonymization in forwarded machine reports | HMAC-SHA256 with a fixed key | `crypto/hmac`, `crypto/sha256` |
| S3 upload credentials (presigned POST policies) | AWS Signature Version 4 (HMAC-SHA256) | `crypto/hmac`, `crypto/sha256` |
| Swift upload credentials (TempURLs) | HMAC-SHA1, the Swift TempURL default | `crypto/hmac`, `crypto/sha1` |
| Server outbound TLS, and admin tool TLS without a custom CA or `--skip-tls-verify` | TLS 1.2 or later (Go default) | `crypto/tls`, `crypto/x509` |
| Admin tool TLS with a custom CA (`--ca` or `LIVEPATCH_CA_CRT`) or `--skip-tls-verify` | TLS 1.3 or later | `crypto/tls`, `crypto/x509` |

**External packages providing cryptographic functionality:**

- [`gopkg.in/macaroon.v2`](https://gopkg.in/macaroon.v2): Macaroon creation and verification
- [`github.com/go-macaroon-bakery/macaroon-bakery`](https://github.com/go-macaroon-bakery/macaroon-bakery): Higher-level macaroon bakery with public key cryptography
- [`golang.org/x/crypto`](https://golang.org/x/crypto): bcrypt, NaCl box/secretbox, curve25519
- Go standard library `crypto/*`: TLS, HMAC, SHA-1, SHA-256, X.509

The server does not encrypt its database, configuration, or patch storage. Use encryption at rest provided by your hosts, database, and storage services; see {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`.

### Data in transit

TLS for clients and the admin tool is terminated by the frontend, not the server. Enforcement varies by path:

| Communication path | TLS |
| :---- | :---- |
| Livepatch Server to Livepatch Client | Optional (operator-configured) |
| Livepatch Server to Livepatch Admin Tool | Optional (operator-configured) |
| Frontend to Livepatch Server | Not supported; keep on a trusted network |
| Livepatch Server to Canonical hosted Livepatch service | Optional (operator-configured) |
| Livepatch Server to Ubuntu Pro service | Optional (HTTPS `contracts.url`) |
| Livepatch Server to PostgreSQL | Optional (operator-configured) |
| Livepatch Server to S3 | Optional (`patch-storage.s3-secure`) |
| Livepatch Server to Swift | Optional (operator-configured) |
| Livepatch Server to Google Cloud Storage or Azure Blob Storage | Optional (HTTPS endpoints) |

## Security support

The Livepatch Server is released as a Kubernetes charm, a machine charm, and a standalone snap. Releases are made ad-hoc as features and bug fixes are implemented. When a security vulnerability in an upstream dependency is detected, a best-effort attempt is made to upgrade the dependency to the latest version that fixes the vulnerability. This security fix is then included in the next ad-hoc release.

A fix for a reported security issue that is directly present in the on-premises Livepatch Server can be prioritized and released as a minor version upgrade for the charms and snap.

Security fixes are delivered on `latest/stable`; see {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>`. Canonical customers can obtain support through the [Canonical support portal](https://portal.support.canonical.com/).

## Related security documentation

- {ref}`Authentication reference <server-reference-livepatch-server-authentication>`: Technical details on all authentication and authorization mechanisms.
- {ref}`Configure the server securely <server-how-to-guides-configure-livepatch-server-securely>`: Initial configuration and reset procedures.
- {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`: Guidance on securely configuring the server for production deployments.
- {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>`: Keeping deployments up to date.
- {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`: Storage protection, backup, and restore.
- {ref}`Integrate with other services securely <server-how-to-guides-integrate-services-securely>`: Securing connections to supporting services.
- {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`: How to use the security event logging and Prometheus monitoring features.
- {ref}`Logging schema <server-reference-logging-schema>`: Structured log fields and security-event identifiers.
- {ref}`Decommission the server securely <server-how-to-guides-decommission-livepatch-server-securely>`: How to securely remove the server and its data.
- {ref}`Security lifecycle <server-explanation-security-lifecycle>`: How security updates are delivered and applied.
- {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`: How to report security vulnerabilities.