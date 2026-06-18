---
myst:
  html_meta:
    description: "Patch Security - technical reference for Livepatch client."
---


(client-reference-patch-security)=

# Patch Security

This document acts as a reference on how patches are secured and outlines the cryptography used by the Livepatch client for this purpose.

## Patch Verification

Patches are downloaded in a multi-step process,

1. Client queries for any new patches from the Livepatch-server.
2. The server returns a SHA256 checksum of the patch contents and a link to the patch file.
3. The client downloads the patch and verifies that a checksum of the downloaded patch matches the expected value.

This process ensures the integrity of the downloaded file using SHA256 hashing.

Go packages used for patch verification:
- crypto/sha256

## Patch Signatures

Patch files are distributed as tarballs. Within each tarball is some metadata and a Linux kernel module (.ko file). The kernel module is responsible for modifying the running kernel to patch high and critical vulnerabilities.

All kernel modules are signed by Canonical to verify their authenticity. This process is done using asymmetric encryption.

- Signature algorithm: SHA512 with RSA
- Canonical’s [Public key](https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/jammy/plain/debian/certs/canonical-livepatch-all.pem)

Kernel modules are authenticated before they are installed, ensuring that the patch was made by Canonical and securing the Livepatch client against installation of maliciously crafted patches. The public keys used for signature verification are baked into the Livepatch client, and therefore, no external access is required to use them.

Go packages used in the process of verifying kernel module signatures:

- encoding/pem for decoding PEM encoded public certificates
- crypto/x509 for parsing the public certificates
- github.com/smallstep/pkcs7 for parsing and verifying the signature with the public certificates.

## TLS communication

The Livepatch client supports HTTPS as a transport layer protocol. This relies on TLS communication. Without delving into TLS, the Livepatch client uses certificates from the host machine’s CA cert pool along with additional fixed certificates, to verify the authenticity of the Livepatch server. The livepatch client also provides users with the ability to add their custom certificates to the CA cert pool.

The client supports a minimum of TLS v1.2.

The `remote-server` config option on the client influences the upstream Livepatch server. There is no client side enforcement that TLS be used, and an on-premises deployment of the Livepatch server may decide to forgo TLS although this is not recommended. The Canonical hosted Livepatch server redirects HTTP traffic to HTTPS.

The Livepatch client provides users with the ability to add their custom certificates to the CA cert pool. For more information on how to add custom certificates to the CA certificate pool, see the [configuration reference](/client/reference/platform/configuration-options.md) for the Livepatch client. The custom CA certificates should be encoded as PEM.

The Livepatch client can also connect to the hosted Livepatch server or an on-premises Livepatch server through HTTPS proxies. See [this how-to guide](/client/how-to-guides/configuration/configure-proxy.md) to understand how to configure proxies for secure communication.

The Livepatch client can also be configured to enforce TLS for all patch downloads. The Canonical-hosted Livepatch server will always serve a `https` URL, so that all patch downloads from the hosted patch storage are made securely over TLS. For more information on how to enforce TLS patch downloads, when clients are getting updates from on-premises Livepatch servers, see the [configuration reference](/client/reference/platform/configuration-options.md) for the Livepatch client.

Go packages used for setting up TLS configuration for the Livepatch client:
- crypto/tls
