---
myst:
  html_meta:
    description: "Technical reference for Livepatch Client patch security, covering patch verification via SHA-256 checksums, RSA-signed kernel modules, and TLS communication."
---

(client-reference-patch-security)=

# Patch security

This document describes how live kernel patches are secured and outlines the cryptographic mechanisms used by the Livepatch Client.

## Patch verification

Patches are downloaded in a multi-step process:

1. The client queries the Livepatch Server for available patches.
2. The server returns a SHA-256 checksum of the patch contents and a URL to the patch file.
3. The client downloads the patch and verifies that the SHA-256 checksum of the downloaded file matches the expected value.

This process ensures file integrity using SHA-256 hashing.

Go packages used for patch verification:

- [`crypto/sha256`](https://pkg.go.dev/crypto/sha256)

## Patch signatures

Patch files are distributed as tarballs containing metadata and a Linux kernel module (`.ko` file). The kernel module modifies the running kernel to patch high and critical vulnerabilities.

All kernel modules are signed by Canonical to verify their authenticity, using asymmetric encryption:

- Signature algorithm: SHA-512 with RSA
- Canonical's [public key](https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/jammy/plain/debian/certs/canonical-livepatch-all.pem)

Kernel modules are authenticated before they are installed. This ensures the patch was produced by Canonical and protects the client from installing maliciously crafted patches. The public keys used for signature verification are embedded in the Livepatch Client and do not require external access.

Go packages used for verifying kernel module signatures:

- [`encoding/pem`](https://pkg.go.dev/encoding/pem) — Decoding PEM-encoded public certificates
- [`crypto/x509`](https://pkg.go.dev/crypto/x509) — Parsing public certificates
- [`github.com/smallstep/pkcs7`](https://github.com/smallstep/pkcs7) — Parsing and verifying signatures with public certificates

## TLS communication

The Livepatch Client supports HTTPS as its transport protocol, relying on standard TLS communication. The client uses certificates from the host machine's CA certificate pool, together with additional fixed certificates, to verify the authenticity of the Livepatch Server. Custom CA certificates can also be added to the pool.

The client requires a minimum of TLS v1.2.

The `remote-server` configuration option determines which upstream Livepatch Server the client contacts. There is no client-side enforcement that TLS be used — an on-premises Livepatch Server may operate without TLS, although this is not recommended. Canonical's hosted Livepatch Server redirects HTTP traffic to HTTPS.

To add custom CA certificates to the CA certificate pool, see the [Livepatch Client configuration reference](/client/reference/platform/configuration-options.md). Custom CA certificates must be encoded as PEM.

The Livepatch Client can connect to the hosted Livepatch Server or an on-premises server through HTTPS proxies. See the [proxy configuration guide](/client/how-to-guides/configuration/configure-proxy.md) for instructions on configuring proxy communication.

The Livepatch Client can also be configured to enforce TLS for all patch downloads. When connecting to Canonical's hosted service, the server always provides an `https` URL, ensuring patch downloads occur over TLS. To enforce TLS for patch downloads from an on-premises Livepatch Server, see the [configuration reference](/client/reference/platform/configuration-options.md).

Go packages used for TLS configuration:

- [`crypto/tls`](https://pkg.go.dev/crypto/tls)