---
myst:
  html_meta:
    description: "Overview of Livepatch client security including patch verification, cryptographic signatures, TLS communication, and related security documentation."
---

(client-explanation-security-overview)=

# Security overview

This document provides an overview of the security features and practices implemented in the Livepatch client. It describes the built-in protections and points to more detailed documentation for specific cryptographic approaches and operational guides.

## Built-in protections

The Livepatch client incorporates several mechanisms to ensure the integrity and authenticity of patches and the security of communication:

- **Patch verification**: Patches undergo verification using SHA256 checksums to confirm file integrity after download. For more details, see the [Patch security reference](/client/reference/patches/patch-security.md).

- **Patch signatures**: All Linux kernel modules distributed as patches are cryptographically signed by Canonical using SHA512 with RSA. This verifies authenticity and prevents the installation of malicious patches. For more information, see the [Patch security reference](/client/reference/patches/patch-security.md).

- **TLS communication**: The Livepatch client uses HTTPS with TLS v1.2 or higher for secure communication with the Livepatch server, using host machine CA certificates and additional fixed certificates for server authentication. For details on TLS communication, see the [Patch security reference](/client/reference/patches/patch-security.md).

## Risks

While the Livepatch client is designed with security in mind, be aware of the following potential risks:

- **TLS enforcement (on-premises deployments)**: The Canonical hosted Livepatch server redirects HTTP to HTTPS, but there is no client-side enforcement for TLS usage in on-premises deployments. Forgoing TLS in such deployments is not recommended.

- **Privileged access to auth tokens**: Authentication for the Livepatch client can be established using [Auth tokens](/server/reference/authentication/authentication.md#client-apis) or [Resource tokens](/server/reference/authentication/authentication.md#client-apis). After successful client registration, a machine token is issued to the client for subsequent authentication. Control privileged access to this token to prevent use by a malicious third party.

- **Data at rest**: The Livepatch client does not encrypt its data at rest. All client data is stored in the `$SNAP_COMMON` and `$SNAP_DATA` directories. The Livepatch client snap ensures only the root user has read and write access to sensitive data in these directories. Maintain strict privileged access controls to prevent unauthorized access to snap data.

## Related documentation

- [Patch security](/client/reference/patches/patch-security.md): In-depth details on the cryptographic approaches used by the Livepatch client for patch verification, signatures, and secure communication.
- [Secure client operation](/client/how-to-guides/security/securely-configure-and-operate-the-client.md): How to securely configure and operate the Livepatch client.
- [Secure client decommissioning](/client/how-to-guides/security/securely-decommission-the-client.md): How to securely decommission the Livepatch client snap and its data.
- [Security lifecycle](/client/explanation/security/security-lifecycle.md): How the security of the Livepatch client is maintained, with guidelines on controlling security updates to the snap.
- [Reporting security vulnerabilities](/client/how-to-guides/security/report-client-vulnerability.md): The process for reporting security vulnerabilities in the Livepatch client, including information about the Ubuntu Security disclosure and embargo policy.
