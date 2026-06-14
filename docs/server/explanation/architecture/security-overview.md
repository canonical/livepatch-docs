---
myst:
  html_meta:
    description: "Security Overview - learn about this topic in Livepatch on-prem."
---


(server-explanation-security-overview)=

# Security Overview

This document provides an overview of the security features and practices implemented within the on-premises Livepatch Server. It aims to give operators a general perspective on how security is handled, outlining built-in protections, authentication mechanisms, and pointing to more detailed documentation for specific cryptographic approaches and operational guides.

The on-premises Livepatch Server is distributed as a Kubernetes charm and a snap package. It synchronizes kernel patches from Canonical's hosted Livepatch service and serves them to Livepatch clients within the operator's infrastructure.

## **System overview from the perspective of security**

The on-premises Livepatch Server acts as the central component in an on-premises Livepatch deployment, coordinating between administrators, clients, Canonical's hosted service, and external services. The following components and trust boundaries are relevant to the security posture of the system:

### Components

- **On-premises Livepatch Server**: The core service that manages patches and serves them to clients within the operator's infrastructure. It exposes HTTP APIs for **Livepatch Client**, **Livepatch Admin Tool**:, and server-to-server communication.
- **PostgreSQL Database**: Stores all persistent data including authentication tokens, patch metadata, machine registrations, and macaroon root keys.
- **Patch Storage Backend**: Stores patch binary files. Supports multiple backends: filesystem, PostgreSQL, OpenStack Swift, and Amazon S3.
- **Livepatch Admin Tool**: CLI tool used by administrators to manage the server (tokens, tiers, patches, webhooks).
- **Livepatch Client**: The endpoint agent that queries the server for patches and applies them to the running kernel.
- **Canonical hosted Livepatch service**: Canonical's hosted Livepatch service from which the on-premises server synchronizes patch data.
- **Ubuntu Pro Service**: External service used to validate Ubuntu Pro resource tokens from clients.

### Trust boundaries

- **Client network to Livepatch Server**: In on-premises deployments, Livepatch clients and the server typically reside on the same internal network. TLS between clients and the server is optional — operators may choose to forgo TLS if the internal network is trusted. When TLS is required, termination is handled by a reverse proxy or Kubernetes ingress controller in front of the server. The charm supports ingress via the `nginx-route` integration (legacy) or the `ingress` integration (modern, using Traefik or Gateway API Integrator).
- **Admin network to Livepatch Server**: Admin API requests should originate from trusted networks. Authentication is enforced through macaroons obtained via Basic Auth.
- **Livepatch Server to PostgreSQL**: Database connections cross a trust boundary. TLS is supported but optional for this connection.
- **Livepatch Server to external services**: Connections to the Ubuntu Pro service enforce TLS. The charm supports configuring a custom CA certificate (`contracts.ca`) for verifying the Pro service certificate, which is installed into the container's system CA trust store. Connections to patch storage backends (S3, Swift) support TLS.
- **Livepatch Server to Pro Airgapped Server**: For air-gapped environments, the charm can integrate with a Pro Airgapped Server via the `pro-airgapped-server` Juju relation to validate resource tokens without external network access.
- **Patch synchronization with Canonical's hosted service**: The on-premises server synchronizes patches from Canonical's hosted Livepatch service using either a contract resource token or a sync token for authentication. Resource tokens are obtained from the Ubuntu Pro contracts service using the `get-resource-token` Juju action (charm) or by setting the contract token via `snap set canonical-livepatch-server token=<CONTRACT_TOKEN>` (snap), which triggers the configure hook to exchange it for a resource token automatically. Sync tokens are issued by the upstream server's admin tool. The active token is configured via `patch-sync.token`. Proxy support is available for environments where direct connectivity is restricted (`patch-sync.proxy.*` configuration).
- **Federation (hierarchical on-premises deployments)**: In federated deployments, a designated on-premises server pulls patches from Canonical's hosted service/ Downstream on-premises servers (for example, in different data centres) pull patches from this designated server. To authorize these connections, the administrator uses the admin tool to issue sync tokens from the upstream server to each downstream server. .

## **Built-in protection**

The Livepatch Server incorporates several mechanisms to ensure the security of its operations:

- **Multi-layered authentication**: The server supports multiple authentication methods depending on the API consumer: macaroons (Basic Auth-backed) for admin users, machine tokens and Ubuntu Pro resource tokens for clients, and contract resource tokens or sync tokens for patch synchronization with upstream servers. For more details, refer to the [Authentication reference](/server/reference/authentication/authentication.md).
- **Request rate limiting**: An HTTP governor limits concurrent request processing and queues excess requests up to a configurable burst limit, protecting against overload and resource exhaustion.
- **Input validation**: All API inputs are validated using strict schemas with regex patterns, length limits, and type checks across 10 validation modules. This mitigates injection attacks and malformed request handling.
- **Structured security event logging**: All authentication successes and failures, authorization decisions, token lifecycle events, admin activity, and system events are logged as structured JSON in compliance with OWASP logging standards.
- **Prometheus monitoring**: The server exposes operational metrics including authentication durations, request latencies, database errors, queue depths, and overload counts through a `/metrics` endpoint.
- **(charm) Log redaction**: The Kubernetes charm includes a log redaction module that scrubs sensitive data from charm logs before they reach any handler. Redacted patterns include database connection URIs with embedded credentials, HTTP authorization headers, key-value pairs containing passwords/tokens/API keys, and specific environment variables carrying secrets (e.g., `LP_CONTRACTS_PASSWORD`, `LP_AUTH_BASIC_USERS`, `LP_DATABASE_CONNECTION_STRING`).
- **(charm) Grafana dashboards**: The charm provides a `metrics-endpoint` integration for Prometheus scraping and a `grafana-dashboard` integration that ships a pre-built dashboard tracking authentication metrics, token usage by type, contracts service errors, and request performance.
- **(charm) Log forwarding**: The charm supports forwarding application logs to Loki via the `log-proxy` (with Promtail, for Juju < 3.4) or `logging` (direct log forwarding, for Juju >= 3.4) integrations, enabling centralized log collection.
- **(charm) Container health checks**: The charm configures a Pebble HTTP health check against the `/debug/info` endpoint, running every 60 seconds, to detect and recover from service failures.
- **(snap) Strict snap confinement**: The snap package uses `strict` confinement, restricting the server to only the `network` and `network-bind` interfaces. This limits the impact of a potential compromise by preventing access to the host system beyond network operations.

## **Risks**

While the Livepatch Server is designed with security in mind, it is important to be aware of potential risks:

- **TLS not natively terminated**: The Livepatch Server does not terminate TLS itself. When TLS is required, it relies on an external reverse proxy, Kubernetes ingress controller, or load balancer for TLS termination. In on-premises deployments where clients and the server are on a trusted internal network, operators may choose to run without TLS. However, if the network is not fully trusted or admin API endpoints are exposed beyond the internal network, configuring TLS termination is strongly recommended to protect authentication credentials in transit.
- **TLS optional for database and storage connections**: Connections between the server and PostgreSQL, S3, and Swift storage backends support TLS but do not enforce it. Running without TLS on these connections in environments with untrusted network segments increases the risk of credential or data interception. Enabling TLS for all backend connections is recommended.
- **No native account lockout**: The server does not implement failed login attempt tracking or temporary lockouts. Admin users are defined in configuration files with bcrypt-hashed passwords. Brute-force protection should be implemented at the network layer (firewall rules, ingress rate limiting).
- **Configuration-based secret management**: Admin credentials and storage credentials are provided through configuration options or environment variables. All secrets are passed to the workload container as environment variables. Operators should use Juju secrets or Kubernetes secrets to protect sensitive configuration options. The charm's log redaction module mitigates the risk of secrets appearing in charm logs, but secrets may still be visible in Pebble plan output or container environment inspection.
- **Privileged access to admin credentials**: Admin Basic Auth passwords are stored as bcrypt hashes in the server configuration. Access to the configuration file or environment variables grants the ability to reconfigure the server. Maintaining strict access controls on configuration files and environment variables is essential.

## **Security of data**

### Data collection and storage

The Livepatch Server collects and stores the following categories of data:

- **Machine registration data**: Machine IDs, authentication tokens, and tier information for registered Livepatch clients.
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

- [gopkg.in/macaroon.v2](https://gopkg.in/macaroon.v2) — Macaroon creation and verification
- [github.com/go-macaroon-bakery/macaroon-bakery](https://github.com/go-macaroon-bakery/macaroon-bakery) — Higher-level macaroon bakery with public key cryptography
- [`golang.org/x/crypto`](http://golang.org/x/crypto) — bcrypt, NaCl box/secretbox, curve25519
- Go standard library `crypto/*` — TLS, HMAC, SHA256, X.509

### Data in transit

TLS encryption is supported for all communication channels but enforcement varies:

| Communication path | TLS |
| :---- | :---- |
| Livepatch Server ↔ Livepatch Client | Optional (operator-configured) |
| Livepatch Server ↔ Livepatch Admin Tool | Optional (operator-configured) |
| Livepatch Server ↔ Canonical hosted Livepatch service | Optional (operator-configured) |
| Livepatch Server ↔ Ubuntu Pro Service | Enforced |
| Livepatch Server ↔ PostgreSQL | Optional (operator-configured) |
| Livepatch Server ↔ S3 | Optional (`s3_secure` flag) |
| Livepatch Server ↔ Swift | Optional (operator-configured) |

TLS is implemented using Go's standard library (`crypto/tls` and `crypto/x509`). The minimum supported TLS version is 1.2.

## **Security support**

The on-premises Livepatch Server is released as a Kubernetes charm and a snap for on-premises use. Releases are made ad-hoc as features and bug fixes are implemented. When a security vulnerability in an upstream dependency is detected, a best-case effort is made to upgrade the dependency to the latest version that fixes the vulnerability. This security fix is then included in the next ad-hoc release.

A fix for a reported security issue that is directly present in the on-premises Livepatch Server can be prioritized and released as a minor version upgrade for both the charm and snap.

## **Related security documentation**

- [Authentication reference](/server/reference/authentication/authentication.md): Technical details on all authentication and authorization mechanisms.
- [Harden your deployment](/server/how-to-guides/security/harden-your-deployment.md): How to securely configure the server for production deployments.
- [Configure logging and monitoring](/server/how-to-guides/operations/configure-logging-and-monitoring.md): How to use the security event logging and Prometheus monitoring features.
- [Decommission the server securely](/server/how-to-guides/operations/decommission.md): How to securely remove the server and its data.
- [Security lifecycle](/server/explanation/architecture/security-lifecycle.md): How security updates are delivered and applied.
- [Report server vulnerabilities](/server/how-to-guides/security/report-server-vulnerabilities.md): How to report security vulnerabilities.
