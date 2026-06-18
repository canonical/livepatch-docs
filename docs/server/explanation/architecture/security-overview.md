---
myst:
  html_meta:
    description: "Explore the security architecture of the Livepatch on-premises server including trust boundaries, authentication, cryptographic mechanisms, and security recommendations."
---

(server-explanation-security-overview)=
# Security overview

> See also: {ref}`Authentication and authorization <server-reference-livepatch-server-authentication>`, {ref}`Security lifecycle <server-explanation-security-lifecycle>`, {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`

The Livepatch on-premises server synchronizes kernel patches from Canonical's hosted Livepatch service and serves them to Livepatch Clients within the operator's infrastructure. It is distributed as a Kubernetes charm and a snap package.

This document provides an overview of the security features and practices implemented in the on-premises Livepatch Server. It describes the system from a security perspective, outlines built-in protections, and references more detailed documentation for specific cryptographic approaches and operational guidance.

## System overview

The on-premises Livepatch Server acts as the central component in an on-premises Livepatch deployment, coordinating between administrators, clients, Canonical's hosted service, and external services.

### Components

The following components are part of a Livepatch on-premises deployment:

- **On-premises Livepatch Server**: The core service that manages patches and serves them to clients within the operator's infrastructure. It exposes HTTP APIs for the Livepatch Client, the Livepatch Admin Tool, and server-to-server communication.
- **PostgreSQL database**: Stores all persistent data including authentication tokens, patch metadata, machine registrations, and macaroon root keys.
- **Patch storage backend**: Stores patch binary files. Supports multiple backends: filesystem, PostgreSQL, OpenStack Swift, and Amazon S3.
- **Livepatch Admin Tool**: CLI tool used by administrators to manage the server (tokens, tiers, patches, webhooks).
- **Livepatch Client**: The endpoint agent that queries the server for patches and applies them to the running kernel.
- **Canonical hosted Livepatch service**: Canonical's hosted Livepatch service from which the on-premises server synchronizes patch data.
- **Ubuntu Pro service**: External service used to validate Ubuntu Pro resource tokens from clients.

### Trust boundaries

The following trust boundaries are relevant to the security posture of the system:

- **Client network to Livepatch Server**: In on-premises deployments, Livepatch Clients and the server typically reside on the same internal network. TLS between clients and the server is optional -- operators may choose to forgo TLS if the internal network is trusted. When TLS is required, termination is handled by a reverse proxy or Kubernetes ingress controller in front of the server. The charm supports ingress via the `nginx-route` integration (legacy) or the `ingress` integration (modern, using Traefik or the Gateway API Integrator).
- **Admin network to Livepatch Server**: Admin API requests should originate from trusted networks. Authentication is enforced through macaroons obtained via Basic Auth.
- **Livepatch Server to PostgreSQL**: Database connections cross a trust boundary. TLS is supported but optional for this connection.
- **Livepatch Server to external services**: Connections to the Ubuntu Pro service enforce TLS. The charm supports configuring a custom CA certificate (`contracts.ca`) for verifying the Pro service certificate, which is installed into the container's system CA trust store. Connections to patch storage backends (S3, Swift) support TLS.
- **Livepatch Server to Pro Airgapped Server**: For air-gapped environments, the charm can integrate with a Pro Airgapped Server via the `pro-airgapped-server` Juju relation to validate resource tokens without external network access.
- **Patch synchronization with Canonical's hosted service**: The on-premises server synchronizes patches from Canonical's hosted Livepatch service using either a contract resource token or a sync token for authentication. Resource tokens are obtained from the Ubuntu Pro contracts service using the `get-resource-token` Juju action (charm) or by setting the contract token via `snap set canonical-livepatch-server token=<CONTRACT_TOKEN>` (snap), which triggers the configure hook to exchange it for a resource token automatically. Sync tokens are issued by the upstream server's admin tool. The active token is configured via `patch-sync.token`. Proxy support is available for environments where direct connectivity is restricted (`patch-sync.proxy.*` configuration).
- **Federation (hierarchical on-premises deployments)**: In federated deployments, a designated on-premises server pulls patches from Canonical's hosted service. Downstream on-premises servers (for example, in different data centers) pull patches from this designated server. To authorize these connections, the administrator uses the admin tool to issue sync tokens from the upstream server to each downstream server.

## Built-in protections

The Livepatch Server incorporates several mechanisms to ensure the security of its operations:

- **Multi-layered authentication**: The server supports multiple authentication methods depending on the API consumer: macaroons (Basic Auth-backed) for admin users, machine tokens and Ubuntu Pro resource tokens for clients, and contract resource tokens or sync tokens for patch synchronization with upstream servers. For more details, refer to the {ref}`Authentication reference <server-reference-livepatch-server-authentication>`.
- **Request rate limiting**: An HTTP governor limits concurrent request processing and queues excess requests up to a configurable burst limit, protecting against overload and resource exhaustion.
- **Input validation**: All API inputs are validated using strict schemas with regex patterns, length limits, and type checks across validation modules. This mitigates injection attacks and malformed request handling.
- **Structured security event logging**: All authentication successes and failures, authorization decisions, token lifecycle events, admin activity, and system events are logged as structured JSON in compliance with OWASP logging standards.
- **Prometheus monitoring**: The server exposes operational metrics including authentication durations, request latencies, database errors, queue depths, and overload counts through a `/metrics` endpoint.
- **(charm) Log redaction**: The Kubernetes charm includes a log redaction module that scrubs sensitive data from charm logs before they reach any handler. Redacted patterns include database connection URIs with embedded credentials, HTTP authorization headers, key-value pairs containing passwords/tokens/API keys, and specific environment variables carrying secrets (for example, `LP_CONTRACTS_PASSWORD`, `LP_AUTH_BASIC_USERS`, `LP_DATABASE_CONNECTION_STRING`).
- **(charm) Grafana dashboards**: The charm provides a `metrics-endpoint` integration for Prometheus scraping and a `grafana-dashboard` integration that ships a pre-built dashboard tracking authentication metrics, token usage by type, contracts service errors, and request performance.
- **(charm) Log forwarding**: The charm supports forwarding application logs to Loki via the `log-proxy` integration (with Promtail, for Juju < 3.4) or the `logging` integration (direct log forwarding, for Juju >= 3.4), enabling centralized log collection.
- **(charm) Container health checks**: The charm configures a Pebble HTTP health check against the `/debug/info` endpoint, running every 60 seconds, to detect and recover from service failures.
- **(snap) Strict snap confinement**: The snap package uses `strict` confinement, restricting the server to only the `network` and `network-bind` interfaces. This limits the impact of a potential compromise by preventing access to the host system beyond network operations.

## Risks

While the Livepatch Server is designed with security in mind, the following risks should be considered:

- **TLS not natively terminated**: The Livepatch Server does not terminate TLS itself. When TLS is required, it relies on an external reverse proxy, Kubernetes ingress controller, or load balancer for TLS termination. In on-premises deployments where clients and the server are on a trusted internal network, operators may choose to run without TLS. However, if the network is not fully trusted or admin API endpoints are exposed beyond the internal network, configuring TLS termination is strongly recommended to protect authentication credentials in transit.
- **TLS optional for database and storage connections**: Connections between the server and PostgreSQL, S3, and Swift storage backends support TLS but do not enforce it. Running without TLS on these connections in environments with untrusted network segments increases the risk of credential or data interception. Enabling TLS for all backend connections is recommended.
- **No native account lockout**: The server does not implement failed login attempt tracking or temporary lockouts. Admin users are defined in configuration files with bcrypt-hashed passwords. Brute-force protection should be implemented at the network layer (firewall rules, ingress rate limiting).
- **Configuration-based secret management**: Admin credentials and storage credentials are provided through configuration options or environment variables. All secrets are passed to the workload container as environment variables. Operators should use Juju secrets or Kubernetes secrets to protect sensitive configuration options. The charm's log redaction module mitigates the risk of secrets appearing in charm logs, but secrets may still be visible in Pebble plan output or container environment inspection.
- **Privileged access to admin credentials**: Admin Basic Auth passwords are stored as bcrypt hashes in the server configuration. Access to the configuration file or environment variables grants the ability to reconfigure the server. Maintaining strict access controls on configuration files and environment variables is essential.

## Security of data

### Data collection and storage

The Livepatch Server collects and stores the following categories of data:

- **Machine registration data**: Machine IDs, authentication tokens, and tier information for registered Livepatch Clients.
- **Patch metadata**: Patch versions, checksums, distribution/architecture mappings, and storage references.
- **Machine reports**: When enabled, the server collects machine status reports from clients. Machine IDs in reports are hashed using HMAC before storage to protect client identity.
- **Admin credentials**: Bcrypt-hashed passwords for Basic Auth admin users.
- **Macaroon root keys**: Stored in PostgreSQL with a 24-hour expiry policy, used for Basic Auth-backed admin session authentication.

All persistent data is stored in PostgreSQL. Patch binary files are stored in the configured patch storage backend (filesystem, PostgreSQL, S3, or Swift).

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
| Machine ID hashing in reports | HMAC | `crypto/hmac` |
| TLS communication | TLS 1.2+ | `crypto/tls`, `crypto/x509` |

**External packages providing cryptographic functionality:**

- [`gopkg.in/macaroon.v2`](https://gopkg.in/macaroon.v2): Macaroon creation and verification
- [`github.com/go-macaroon-bakery/macaroon-bakery`](https://github.com/go-macaroon-bakery/macaroon-bakery): Higher-level macaroon bakery with public key cryptography
- [`golang.org/x/crypto`](https://golang.org/x/crypto): bcrypt, NaCl box/secretbox, curve25519
- Go standard library `crypto/*`: TLS, HMAC, SHA256, X.509

### Data in transit

TLS encryption is supported for all communication channels but enforcement varies:

| Communication path | TLS |
| :---- | :---- |
| Livepatch Server to Livepatch Client | Optional (operator-configured) |
| Livepatch Server to Livepatch Admin Tool | Optional (operator-configured) |
| Livepatch Server to Canonical hosted Livepatch service | Optional (operator-configured) |
| Livepatch Server to Ubuntu Pro service | Enforced |
| Livepatch Server to PostgreSQL | Optional (operator-configured) |
| Livepatch Server to S3 | Optional (`s3_secure` flag) |
| Livepatch Server to Swift | Optional (operator-configured) |

TLS is implemented using Go's standard library (`crypto/tls` and `crypto/x509`). The minimum supported TLS version is 1.2.

## Security support

The on-premises Livepatch Server is released as a Kubernetes charm and a snap for on-premises use. Releases are made ad-hoc as features and bug fixes are implemented. When a security vulnerability in an upstream dependency is detected, a best-effort attempt is made to upgrade the dependency to the latest version that fixes the vulnerability. This security fix is then included in the next ad-hoc release.

A fix for a reported security issue that is directly present in the on-premises Livepatch Server can be prioritized and released as a minor version upgrade for both the charm and snap.

## Related security documentation

- {ref}`Authentication reference <server-reference-livepatch-server-authentication>`: Technical details on all authentication and authorization mechanisms.
- {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`: Guidance on securely configuring the server for production deployments.
- {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`: How to use the security event logging and Prometheus monitoring features.
- {ref}`Decommission the server securely <server-how-to-guides-decommission-livepatch-server-securely>`: How to securely remove the server and its data.
- {ref}`Security lifecycle <server-explanation-security-lifecycle>`: How security updates are delivered and applied.
- {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`: How to report security vulnerabilities.