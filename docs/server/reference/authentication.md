---
myst:
  html_meta:
    description: "Authentication - technical reference for Livepatch on-prem server."
---


(livepatch_on_prem-reference-livepatch-server-authentication)=

# Livepatch Server Authentication

This document provides a technical reference for all authentication and authorization mechanisms used by the on-premises Livepatch Server. It covers the authentication flows for each API surface and the cryptographic technologies involved.

## Admin APIs

These APIs are consumed by administrators through the Livepatch Admin Tool.

The following authentication flows are supported:

- **Basic Auth → Macaroon**: Login using [Basic Auth](#livepatch_on_prem-reference-basic-auth) to obtain a [macaroon](#livepatch_on_prem-reference-macaroons), then use the macaroon to authenticate subsequent requests. Macaroons expire after 240 hours.

All admin activity is logged as a structured security event with the `authz_admin` event code.

(livepatch_on_prem-reference-client-apis)=

## Client APIs

These APIs are consumed by Livepatch Client instances.

- **Auth Token**: A UUIDv4 token used to authenticate and register a machine. After successful registration, a machine token is issued for subsequent requests. This is a legacy token format that is no longer used for new clients.
- **Resource Token**: A macaroon verified against the Ubuntu Pro backend to authenticate requests. This token is provided to the Livepatch Client by the Ubuntu Pro client when enabling Pro.

After authentication, client requests are authorized based on tier and affordances (architecture and kernel compatibility).

## Server APIs

These APIs are used for patch synchronization between on-premises servers and their upstream patch source (Canonical's hosted Livepatch service or another on-premises server).

- **Contract Resource Token**: A resource token obtained from the Ubuntu Pro contracts service, used to authenticate the on-premises server with Canonical's hosted Livepatch service. For charm deployments, obtained via the `get-resource-token` Juju action (`juju run canonical-livepatch-server-k8s/<leader> get-resource-token contract-token=<TOKEN>`). For snap deployments, obtained implicitly by setting the contract token (`sudo snap set canonical-livepatch-server token=<CONTRACT_TOKEN>`); the snap's configure hook exchanges it for a resource token and sets `patch-sync.token` automatically. The process exchanges a contract token for a machine token and then retrieves the resource token with appropriate affordances.
- **Sync Token**: A UUIDv4 generated through the Admin APIs of the upstream server (`livepatch-admin sync-tokens add <tier> <token-name>`). Used in both direct (hosted) and federated (on-premises to on-premises) synchronization. Like client tokens, sync tokens specify which tier patches will be retrieved from. Configured via `patch-sync.token`. Stored in PostgreSQL with verification by direct comparison.

## Technologies

(livepatch_on_prem-reference-basic-auth)=

### Basic Auth

HTTP Basic Authentication using a username and Base64-encoded password. The authentication process extracts credentials from the HTTP `Authorization` header, decodes them, and compares the provided password against a stored bcrypt hash using constant-time comparison.

Go packages used:

- [`encoding/base64`](https://pkg.go.dev/encoding/base64)
- [`crypto/subtle`](https://pkg.go.dev/crypto/subtle)
- [`golang.org/x/crypto/bcrypt`](http://golang.org/x/crypto/bcrypt)

(livepatch_on_prem-reference-macaroons)=

### Macaroons

Macaroons are decentralized authentication tokens that use HMAC for cryptographic signatures and symmetric encryption to encode the scope (caveats) of entitlements.

Operations are performed using `HMAC-SHA256` and `XSalsa20-Poly1305`. Go packages used by the [macaroon package](https://gopkg.in/macaroon.v2):

- [`crypto/hmac`](https://pkg.go.dev/crypto/hmac)
- [`crypto/sha256`](https://pkg.go.dev/crypto/sha256)
- [`golang.org/x/crypto/nacl/secretbox`](http://golang.org/x/crypto/nacl/secretbox)

The higher-level [Macaroon Bakery package](https://github.com/go-macaroon-bakery/macaroon-bakery) adds public key cryptography for macaroon operations.

Operations are performed using `Ed25519` and `XSalsa20-Poly1305`. Go packages used:

- [`golang.org/x/crypto/nacl/box`](http://golang.org/x/crypto/nacl/box)
- [`golang.org/x/crypto/curve25519`](http://golang.org/x/crypto/curve25519)

Macaroon root keys are stored in PostgreSQL with a 24-hour expiry policy. Macaroons themselves expire after 240 hours.

### UUID

UUIDv4 is a 128-bit identifier generated using random number generation, represented as 32 hexadecimal digits in 8-4-4-4-12 format. Used for auth tokens, machine tokens, sync tokens, and request trace IDs.

Go packages used:

- [`github.com/google/uuid`](http://github.com/google/uuid)
