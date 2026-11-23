---
myst:
  html_meta:
    description: "Technical reference for Livepatch on-premises server authentication, covering Basic Auth, macaroons, client tokens, sync tokens, and cryptographic technologies."
---

(server-reference-livepatch-server-authentication)=

# Livepatch Server authentication

This reference distinguishes the credentials used for administration, client registration, client requests, and upstream synchronization. For configuration procedures, see [administrator setup](/server/how-to-guides/security/setup-administration-tool.md), [client registration](/server/how-to-guides/operations/use-livepatch-client-with-on-prem-server.md), and [server chaining](/server/how-to-guides/patch-management/chain-livepatch-servers.md).

## Admin APIs

Admin APIs are consumed by administrators through the Livepatch Admin Tool.

The following authentication flow is supported:

- **Basic Auth → Macaroon**: Authenticate using a configured local administrator username and password to obtain a [macaroon](#server-reference-macaroons). The tool uses this session credential for subsequent requests. Macaroons expire after 240 hours.

Administrator passwords are stored as bcrypt hashes. Both charms support a Juju-secret-backed username-and-hash list; the standalone snap uses its configuration settings. Administrator login is independent of the patch storage backend and upstream synchronization credentials.

Authorized administrative requests generate structured security events with the `authz_admin` event code. See [logging and monitoring](/server/how-to-guides/operations/configure-logging-and-monitoring.md).

(server-reference-client-apis)=

## Client APIs

Client APIs are consumed by Livepatch Client instances.

- **Authorization token (auth token)**: A UUIDv4 credential issued by an administrator of your server for a client group and tier. Clients use it to register with that server. Multiple clients can share one authorization token.
- **Machine token**: A credential issued after successful local registration, used by that client for subsequent requests. Your server validates auth and machine tokens against your deployment's database.

The auth-token workflow works in connected and air-gapped deployments. Clients register with and send subsequent requests to the local Livepatch Server, not to Canonical's hosted Livepatch service. Patch availability is governed by the client's assigned tier and kernel compatibility.

## Server APIs

Server APIs are used for patch synchronisation between on-premises servers and their upstream patch source (Canonical's hosted Livepatch service or another on-premises server).

- **Ubuntu Pro resource token**: Used by an on-premises server synchronizing with Canonical's hosted Livepatch service. The deployment's Ubuntu Pro configuration obtains the resource token for use as its upstream synchronization credential. This is separate from administrator login and local client registration.
- **Sync token**: A UUIDv4 credential issued by an administrator of an upstream on-premises server for a specific tier. A downstream server uses it to authenticate synchronization requests to that upstream. The upstream stores and validates the token in its database.

Configure the upstream address and credential through the [patch-sync settings](/server/reference/platform/configuration.md#patch-sync). Both charms support `patch-sync.token-secret` for a Juju-secret-backed credential; the standalone snap uses `lp.patch-sync.token`. Protect upstream credentials separately from client registration and administrator credentials.

An air-gapped server can receive patches through offline imports instead of external synchronization.

## Technologies

(server-reference-basic-auth)=

### Basic Auth

HTTP Basic Authentication encodes `username:password` in Base64 in the HTTP `Authorization` header. Base64 is encoding, not encryption. The server verifies the supplied password against the configured bcrypt hash.

Use a [TLS frontend](/server/how-to-guides/security/setup-tls.md) to protect credentials crossing untrusted networks.

(server-reference-macaroons)=

### Macaroons

Macaroons are decentralized authentication tokens that use HMAC for cryptographic signatures and symmetric encryption to encode the scope (caveats) of entitlements.

Macaroon mechanisms use HMAC-SHA256 and XSalsa20-Poly1305. Macaroon Bakery mechanisms also use Ed25519 public key cryptography.

Macaroon root keys are stored in PostgreSQL with a 24-hour expiry policy. Macaroons themselves expire after 240 hours.

### UUID

UUIDv4 is a randomly generated 128-bit identifier, represented as hexadecimal digits in 8-4-4-4-12 groups. It is used for authorization tokens, locally issued machine tokens, and sync tokens. Treat token values as secrets even though they have the same textual format as ordinary identifiers.