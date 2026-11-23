---
myst:
  html_meta:
    description: "Explore the security architecture of the Livepatch on-premises server including trust boundaries, authentication, cryptographic mechanisms, and security recommendations."
---

(server-explanation-security-overview)=

# Security overview

> See also: {ref}`Authentication and authorization <server-reference-livepatch-server-authentication>`, {ref}`Security lifecycle <server-explanation-security-lifecycle>`, {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`

The Livepatch on-premises server synchronizes kernel patches from Canonical's hosted Livepatch service and serves them to Livepatch Clients within the operator's infrastructure. It is available as a Kubernetes charm, a machine charm, and a standalone snap.

Clients register with their local Livepatch Server and send subsequent check-ins and reports to that server, not to the upstream Livepatch service. Upstream synchronization and offline patch imports are separate server-side operations.

This document provides an overview of the security features and practices implemented in the on-premises Livepatch Server. It describes the system from a security perspective, outlines built-in protections, and references more detailed documentation for specific cryptographic approaches and operational guidance.

## System overview

The on-premises Livepatch Server acts as the central component in an on-premises Livepatch deployment, coordinating between administrators, clients, Canonical's hosted service, and external services.

![Security architecture showing clients, administrator access, the server deployment, backend services, management access, and optional direct patch transfers across trust boundaries.](/_static/images/server-security-architecture.svg)

The diagram shows logical security contexts, not a required network layout. Dashed boxes mark trust boundaries: crossing one means entering a different network, runtime, or administrative context. Components can share a host or network without sharing the same access permissions. Solid arrows show service communication or management access. Dashed arrows show optional direct patch transfers.

### Components

The following components are part of a Livepatch on-premises deployment:

- **Ingress or reverse proxy**: Receives client and administrator requests before forwarding them to the server. It can terminate TLS and enforce network restrictions and additional request limits. These controls depend on the proxy configuration, not the server binary.
- **On-premises Livepatch Server**: The core service that manages patches and serves them to clients within the operator's infrastructure. It exposes HTTP APIs for the Livepatch Client, the Livepatch Admin Tool, and server-to-server communication.
- **PostgreSQL database**: Stores authentication tokens, patch metadata, machine registrations, reports, and macaroon root keys.
- **Patch storage backend**: Stores patch binary files. Supported {ref}`backends <server-reference-patch-storage>` include filesystem, PostgreSQL, Amazon S3, OpenStack Swift, Google Cloud Storage, and Azure Blob Storage.
- **Livepatch Admin Tool**: CLI tool used by administrators to manage the server (tokens, tiers, patches, webhooks).
- **Livepatch Client**: The endpoint agent that queries the server for patches and applies them to the running kernel.
- **Canonical hosted Livepatch service**: Canonical's hosted Livepatch service from which the on-premises server synchronizes patch data.
- **Ubuntu Pro service**: Optional upstream entitlement service used by the on-premises server when synchronizing with Canonical's hosted service. Livepatch Clients connect to the on-premises server, not this service.
- **Management infrastructure**: Juju manages both VM and Kubernetes deployments. Host administrators manage standalone snap deployments. These management paths can change configuration, access credentials, or replace the workload.
- **Observability services**: Optional Prometheus, Grafana, and Loki integrations collect metrics and logs for monitoring and investigation.

Patch transfers do not always pass through the server. When configured to redirect downloads, the server returns a URL from which the client downloads the patch directly. For supported cloud storage backends, the admin tool can also upload patches directly using temporary credentials issued by the server. The storage endpoint and its permissions therefore form a separate security boundary. See {ref}`Patch storage <server-reference-patch-storage>` for these credential models.

### Trust boundaries

TLS is optional within a trusted internal network. Across untrusted networks, TLS is recommended to protect credentials, requests, and responses from interception or modification. Network trust does not remove the need for authentication and authorization.

| Boundary | Security properties and configuration |
| :---- | :---- |
| Client network to server APIs | Clients register using authorization tokens issued by the local server, then authenticate subsequent requests with machine tokens. Tier and affordance checks restrict access to compatible patches. A TLS-terminating ingress or reverse proxy protects traffic when the connection uses HTTPS. |
| Administrator network to admin APIs | Administrators use Basic Auth to obtain macaroons for subsequent requests. Limit admin API access to trusted networks and protect the credentials and token files on administrator machines. See the {ref}`authentication reference <server-reference-livepatch-server-authentication>`. |
| Ingress or reverse proxy to server | The server does not terminate TLS. After TLS termination, the proxy forwards HTTP to the server. Keep that hop within a trusted network and restrict direct access to the backend listener so requests cannot bypass proxy-level controls. For Kubernetes, the charm supports the legacy `nginx-route` integration and the modern `ingress` integration with Traefik or the Gateway API Integrator. |
| Server to database and patch storage | Database connections and remote storage requests use credentials with access to persistent data. PostgreSQL TLS is supported but optional. Configure backend transport security and permissions independently; filesystem storage instead relies on host and directory access controls. See {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`. |
| Server to upstream patch source | Synchronization uses a contract resource token or sync token configured through `patch-sync.token`. Configure the upstream URL and any proxy deliberately, using HTTPS across untrusted networks. The {ref}`authentication reference <server-reference-livepatch-server-authentication>` describes obtaining these tokens. |
| Management infrastructure to workload | Juju and host administration can change the workload and its security settings. Restrict management access separately from client API access; application authentication does not protect against a privileged management administrator. |
| Server to observability services | Metrics and logs leave the workload when monitoring or forwarding integrations are enabled. Restrict monitoring endpoint access and protect the receiving services. Log redaction in the charm is not a guarantee that every application log or exported metric contains no sensitive information. |

In {ref}`federated deployments <server-how-to-guides-chain-on-prem-servers>`, each downstream server crosses the upstream server's API boundary using a sync token issued by that upstream server. Apply the same network and transport protections to this connection as to other server API connections.

In air-gapped deployments, clients register using {ref}`auth tokens issued by the on-premises server <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>` and receive machine tokens validated within the deployment. Patches enter through an [offline transfer workflow](/server/how-to-guides/patch-management/distribute-patches-offline.md). The transfer workstation or media and internal services remain separate trust contexts even when no public network connection exists.

## Built-in protections

The following controls help protect the system. Built-in mechanisms still depend on appropriate deployment configuration; optional integrations are not enabled merely by installing the server.

| Control | Threat mitigated | Availability and operator action |
| :---- | :---- | :---- |
| Authentication and authorization | Unauthorized administration, client access, or patch synchronization. | The server provides different token mechanisms for each API consumer, with tier and affordance checks for clients. Configure admin authentication and issue appropriate tokens using the {ref}`authentication reference <server-reference-livepatch-server-authentication>` and {ref}`admin setup guide <server-how-to-guides-how-to-setup-livepatch-on-prem-server-administration-tool>`. |
| Input validation | Injection attempts and malformed API requests. | API validation uses schema, format, length, and type checks. Validation does not replace authentication or request limits. |
| HTTP request governor | Resource exhaustion from too many concurrent requests. | The server limits concurrent processing and queues excess requests within configured limits. Operators must {ref}`tune the limits and restrict network access <server-how-to-guides-harden-your-livepatch-server-deployment>`; application limits do not protect an already saturated network link. |
| Structured security event logging | Undetected unauthorized access or administrative changes. | The server records authentication, authorization, token lifecycle, administrative, and system events. Operators must retain and review logs; logging alone does not prevent an attack. See {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`. |
| Prometheus metrics | Undetected authentication failures, overload, or backend errors. | The `/metrics` endpoint exposes operational metrics. Operators must restrict access, configure collection, and arrange monitoring and alerting. |
| Charm log redaction | Credential disclosure through charm operator logs. | Both the machine and Kubernetes charms replace recognized credentials, authorization headers, and secret values with `***REDACTED***` in their operator logs. This does not redact the server application's logs or protect secrets exposed through workload inspection. |
| Kubernetes charm observability integrations | Loss of visibility when logs and metrics remain on individual units. | Optional `metrics-endpoint` and `grafana-dashboard` integrations provide Prometheus collection and a dashboard. The `logging` integration forwards logs to Loki. Follow the {ref}`logging and monitoring guide <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>` to configure and secure these integrations and their receiving services. |
| Kubernetes charm health checks | Prolonged service failure. | A Pebble HTTP health check probes `/debug/info` every 60 seconds to detect and recover from service failures. A successful health check is not proof that the deployment is securely configured. |
| Strict snap confinement | Host access following compromise of the server process. | The standalone snap and the snap managed by the machine charm use strict confinement. The server's network interfaces are `network` and `network-bind`; host access remains constrained by snap permissions. Operators must still protect the host and the snap's writable data. |

TLS termination, backend transport security, firewall rules, Kubernetes network policies, and management-plane access controls require operator configuration. The {ref}`hardening guide <server-how-to-guides-harden-your-livepatch-server-deployment>` explains how to configure these protections, while the {ref}`configuration reference <server-reference-configuration>` describes the corresponding server options.

## Risks

While the Livepatch Server is designed with security in mind, the following risks should be considered:

- **TLS not natively terminated**: The Livepatch Server does not terminate TLS itself. When TLS is required, it relies on an external reverse proxy, Kubernetes ingress controller, or load balancer for TLS termination. In on-premises deployments where clients and the server are on a trusted internal network, operators may choose to run without TLS. However, if the network is not fully trusted or admin API endpoints are exposed beyond the internal network, configuring TLS termination is strongly recommended to protect authentication credentials in transit. See {ref}`Set up TLS <server-how-to-guides-how-to-setup-tls-for-livepatch-on-prem>`.
- **TLS optional for database and storage connections**: Connections between the server and PostgreSQL, S3, and Swift storage backends support TLS but do not enforce it. Running without TLS on these connections in environments with untrusted network segments increases the risk of credential or data interception. Enabling TLS for all backend connections is recommended. See {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>` for backend transport settings.
- **No native account lockout**: The server does not implement failed login attempt tracking or temporary lockouts. Admin users are configured with bcrypt-hashed passwords. Brute-force protection should be implemented at the network layer through firewall rules and ingress rate limiting, as described in the {ref}`hardening guide <server-how-to-guides-harden-your-livepatch-server-deployment>`.
- **Configuration-based secret management**: Admin and storage credentials are supplied through configuration options, environment variables, or supported secret references. Restrict access to configuration and workload environments, and use Juju secrets where supported. The charm's log redaction reduces credential exposure in charm logs, but does not protect values visible through workload inspection. For a machine-charm example, see [Move credentials to Juju secrets](/server/how-to-guides/deployment/migrate-to-the-new-machine-charm.md#move-credentials-to-juju-secrets).
- **Privileged access and incorrect access controls**: Access to admin APIs, configuration, or management infrastructure can allow an unauthorized operator to change patch availability, issue tokens, or expose stored data. Protect configuration files and environment variables, restrict admin API and Juju access to authorized operators, and limit database and storage permissions to those needed by the deployment. See {ref}`Authentication <server-reference-livepatch-server-authentication>` and {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`.
- **Unvalidated integration input**: The server validates API inputs, but this does not protect custom integration code or deployment automation from unsafe input handling. Using untrusted values to construct commands, URLs, or requests can cause injection attacks or unintended operations in those systems. Validate and constrain inputs in integrating systems, and use the expected types and formats in the {ref}`configuration reference <server-reference-configuration>`.
- **Resource exhaustion**: Excessive requests, uncontrolled retries, or patch transfers can exhaust application capacity, database connections, storage, or network bandwidth and delay patch delivery. The HTTP governor limits application request processing, but does not protect an already saturated network or every connected backend. Configure request limits and network restrictions using the {ref}`hardening guide <server-how-to-guides-harden-your-livepatch-server-deployment>`, and {ref}`monitor overload and backend errors <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.
- **Incorrect handling of stored or exported data**: Reports, configuration exports, backups, and logs can disclose credentials or information about the managed fleet if shared or retained without appropriate access controls. Machine reports contain client identifiers and patch status information. Restrict access to these artifacts and apply your organization's retention and deletion policies. For information collected by your local server, see {ref}`Machine reports <server-reference-machine-reports>`. If you configure your server to synchronize with Canonical, also review {ref}`Data sent to Canonical servers <server-reference-data-sent-to-canonical-servers>`.
- **Missing audit records and shared identities**: Without retained security logs, unauthorized changes can go unnoticed and administrative actions can be difficult to attribute. Shared administrator identities further reduce accountability. Use individual administrator accounts and a centralized log aggregator to collect security events for investigation, with appropriate access restrictions and retention. See {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>` for log-forwarding integrations and the risks of suppressing security logs.

## Security of data

### Data collection and storage

The Livepatch Server stores data for your deployment and communicates with its configured upstream to synchronize patches. In an air-gapped deployment, patches can instead be imported offline without direct communication with Canonical.

The server handles the following categories of data:

- **Client registration and tokens**: Auth tokens used for registration, machine tokens issued to registered clients, machine IDs, and tier assignments.
- **Sync tokens**: Tokens issued by your server to allow downstream servers to retrieve patches.
- **Patch metadata**: Patch versions, checksums, distribution/architecture mappings, and storage references.
- **Machine reports**: When you enable database reporting, your server stores client machine status reports, including machine identifiers and patch status.
- **Admin credentials**: Bcrypt-hashed passwords for Basic Auth admin users, supplied through deployment configuration or supported Juju secrets.
- **Macaroon root keys**: Stored in PostgreSQL with a 24-hour expiry policy, used for Basic Auth-backed admin session authentication.

Client registration records, auth tokens, machine tokens, sync tokens issued by your server, patch metadata, stored machine reports, and macaroon root keys are kept in PostgreSQL. Deployment configuration, including upstream synchronization credentials and storage credentials, is managed by the snap, charm, or workload environment. Patch binary files use your configured {ref}`patch storage backend <server-reference-patch-storage>`: filesystem, PostgreSQL, Amazon S3, OpenStack Swift, Google Cloud Storage, or Azure Blob Storage.

With the documented {ref}`auth-token workflow <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`, clients register with your server using an auth token and receive machine tokens for subsequent requests. Your server validates these tokens against your deployment's database.

Machine reports sent by clients go to the server each client is configured to use. Clients connected to an on-premises deployment send reports to that Livepatch Server, not directly to Canonical's hosted Livepatch service. This client-to-server reporting is distinct from the server's upstream synchronization. Enabling local database reporting does not enable forwarding of detailed machine reports upstream. See {ref}`Machine reports <server-reference-machine-reports>` for reporting information.

During synchronization, the server sends its deployment ID, machine-count information, and patch-selection information to its upstream. That upstream can be Canonical's hosted service or another on-premises server. Detailed machine-report forwarding is a separate opt-in setting. See {ref}`Data sent to Canonical servers <server-reference-data-sent-to-canonical-servers>` for the information exchanged and the distinction between local reporting and upstream forwarding.

An air-gapped deployment can use local auth and machine tokens, {ref}`import patches offline <server-how-to-guides-distribute-patches-offline>`, and store reports within your infrastructure. This workflow does not require the air-gapped server to contact Canonical. Any upstream servers, storage, and observability services used within the air gap must be available inside your isolated environment.

### Data lifecycle, retention, and deletion

Database storage of machine reports is disabled by default. When enabled, the default configured retention period is **10 days for the machine and Kubernetes charms**, and **7 days for the standalone snap**. The `machine-reports.database.retention-days` setting controls this period. Setting it to `0` disables automatic report cleanup; it does not delete reports immediately.

Automatic cleanup runs periodically on the leader and removes eligible reports in bounded batches. Reports can remain beyond the configured retention period if cleanup falls behind or fails. Database reporting must remain enabled for automatic cleanup to run. Disabling it stops new report writes and automatic cleanup, but does not purge previously stored reports. See the [machine reports configuration options](/server/reference/platform/configuration.md#machine-reports) for the cleanup interval, batch limit, and deployment-specific defaults. Check your deployed configuration rather than assuming defaults apply to an existing or migrated deployment.

Machine-report cleanup does not revoke tokens or delete client registrations, patch metadata, patch binary files, or deployment configuration. These data categories require separate retention and deletion decisions.

Define and enforce retention and deletion policies for your deployment's data, including logs, exported reports, configuration exports, backups, and snapshots. Manage copies held in external storage or observability services separately. Local report cleanup does not remove copies you have already exported or forwarded to an upstream server. For removal of your deployment and its associated data, see {ref}`Decommission a Livepatch Server securely <server-how-to-guides-decommission-livepatch-server-securely>`.

### Cryptographic mechanisms

The following cryptographic technologies are used by the Livepatch Server:

**Authentication:**

| Mechanism | Algorithm |
| :---- | :---- |
| Basic Auth password verification | bcrypt |
| Macaroons (HMAC signatures) | HMAC-SHA256, XSalsa20-Poly1305 |
| Macaroon Bakery (public key crypto) | Ed25519, XSalsa20-Poly1305 |
| Token generation (UUIDv4) | Random number generation |

**Data protection:**

| Mechanism | Algorithm |
| :---- | :---- |
| Outbound TLS connections | TLS 1.2+ |

### Data at rest

Password hashing does not encrypt the database, deployment configuration, report exports, or patch storage. Use encryption at rest provided by your hosts, database deployment, and storage services, and restrict access to their credentials and encryption keys. Apply the same protections to backups, snapshots, and exported data.

See {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>` for secure database export and restore, coordinated patch-storage copies, and recovery-copy retention. See {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>` for host and data-service access controls.

### Data in transit

The Livepatch Server serves HTTP and does not terminate inbound TLS. To protect client and administrator connections with HTTPS, configure TLS termination on an ingress, reverse proxy, or load balancer. Keep the separate HTTP connection from that frontend to the server within a trusted network, or protect it at the network layer.

Configure transport protection for each communication path:

| Communication path | TLS |
| :---- | :---- |
| Livepatch Client to Livepatch frontend | TLS termination at the operator-configured frontend. |
| Livepatch Admin Tool to Livepatch frontend | TLS termination at the operator-configured frontend. |
| Livepatch Server to configured patch upstream | Use an HTTPS upstream address when crossing untrusted networks. |
| Standalone Livepatch Server snap to PostgreSQL | Configure TLS and certificate validation in the database connection string. |
| Livepatch Server to Amazon S3 | Enable HTTPS with `patch-storage.s3-secure=true`. |
| Livepatch Server to OpenStack Swift | Use HTTPS authentication and object-storage endpoints. |
| Livepatch Server to Google Cloud Storage | Use HTTPS endpoints. |
| Livepatch Server to Azure Blob Storage | Use HTTPS endpoints, including in custom connection strings. |

Frontend TLS parameters and certificate management belong to the chosen ingress or proxy. They are separate from TLS used by the server's outbound connections. See {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>` and {ref}`Set up TLS <server-how-to-guides-how-to-setup-tls-for-livepatch-on-prem>` for configuration guidance.

## Security support

The Livepatch Server is released as a Kubernetes charm, a machine charm, and a standalone snap. Operators are expected to keep their deployed charm or standalone snap updated to the current revision in `latest/stable`.

See {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>` for version checks, update procedures, automatic-update controls, and release monitoring.

Releases are made ad-hoc as features and bug fixes are implemented. When a security vulnerability in an upstream dependency is detected, a best-effort attempt is made to upgrade the dependency to the latest version that fixes the vulnerability. This security fix is then included in the next ad-hoc release.

A fix for a reported security issue that is directly present in the on-premises Livepatch Server can be prioritized and released as a minor version upgrade for the charms and snap.

Canonical customers can obtain technical support for the Livepatch Server through the [Canonical support portal](https://portal.support.canonical.com/). See [Get more help](/support/get-more-help.md) for support-ticket guidance. To report a potential vulnerability, follow {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`.

## Related security documentation

- {ref}`Configure the server securely <server-how-to-guides-configure-livepatch-server-securely>`: Pre-deployment checks, initial configuration, and separate configuration-only and full reset procedures.
- {ref}`Set up administrator authentication <server-how-to-guides-how-to-setup-livepatch-on-prem-server-administration-tool>`: Local administrator accounts, protected credential configuration, and credential changes.
- {ref}`Register clients with your server <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`: Local authorization-token registration and machine-token authentication.
- {ref}`Chain on-premises servers <server-how-to-guides-chain-on-prem-servers>`: Upstream synchronization credentials and tier selection.
- {ref}`Authentication reference <server-reference-livepatch-server-authentication>`: Credential types and authentication flows for administration, clients, and synchronization.
- {ref}`Harden your deployment <server-how-to-guides-harden-your-livepatch-server-deployment>`: Guidance on securely configuring the server for production deployments.
- {ref}`Get security updates and support <server-how-to-guides-get-security-updates-and-support>`: Manual updates, standalone snap automatic-update controls, release checks, and support.
- {ref}`Manage data at rest <server-how-to-guides-manage-data-at-rest>`: Storage protection, secure database backup and restore, patch migration, and recovery-copy retention.
- {ref}`Integrate with other services securely <server-how-to-guides-integrate-services-securely>`: Connection protections, deployment-specific network requirements, and existing synchronization and request controls.
- {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`: How to use the security event logging and Prometheus monitoring features.
- {ref}`Logging schema <server-reference-logging-schema>`: Structured log fields, security-event identifiers, and optional request context.
- {ref}`Decommission the server securely <server-how-to-guides-decommission-livepatch-server-securely>`: How to securely remove the server and its data.
- {ref}`Security lifecycle <server-explanation-security-lifecycle>`: How security updates are released and delivered for each distribution.
- {ref}`Report server vulnerabilities <server-how-to-guides-report-a-livepatch-server-vulnerability>`: How to report security vulnerabilities.