---
myst:
  html_meta:
    description: "Security Overview - learn about this topic in Livepatch client."
---


(livepatch-explanation-security-overview)=

# Security Overview

This document provides an overview of the security features and practices implemented within the Livepatch Client. It aims to give users a general perspective on how security is handled, outlining built-in protections and pointing to more detailed documentation for specific cryptographic approaches and operational guides.

## Built-in Protection

The Livepatch client incorporates several mechanisms to ensure the integrity and authenticity of patches and the security of communication:

- **Patch Verification**: Patches undergo a verification process using SHA256 checksums to ensure file integrity upon download. For more details, refer to the [Cryptographic Documentation](/livepatch/reference/patch-security.md).

- **Patch Signatures**: All Linux kernel modules distributed as part of patches are cryptographically signed by Canonical using SHA512 with RSA. This verifies their authenticity and prevents the installation of malicious patches. Further information can be found in the [Cryptographic Documentation](/livepatch/reference/patch-security.md).

- **TLS Communication**: The Livepatch client uses HTTPS and relies on TLS v1.2 or higher for secure communication with the Livepatch server, utilizing host machine CA certificates and additional fixed certificates for server authentication. Details on TLS communication are available in the [Cryptographic Documentation](/livepatch/reference/patch-security.md).

## Risks

While the Livepatch client is designed with security in mind, it is important to be aware of potential risks:

- **TLS Enforcement (On-premises deployments)**: Although the Canonical hosted Livepatch server redirects HTTP to HTTPS, there is no client-side enforcement for TLS usage in on-premises deployments. Forgoing TLS in such scenarios is not recommended due to security implications.

- **Privileged access to Auth Tokens**: Authentication for the Livepatch Client, with the Livepatch Server, can be established using [Auth Tokens](/livepatch_on_prem/reference/authentication.md#client-apis) or [Resource Tokens](/livepatch_on_prem/reference/authentication.md#client-apis). After successful client registration, a machine token is issued to the Livepatch client which can be used for subsequent authentication. Privileged access to this token must be controlled to prevent usage by a malicious third-party.

- **Data at Rest**: The Livepatch client does not encrypt its data at rest. However, all client data is stored in the [$SNAP_COMMON](https://snapcraft.io/docs/reference/development/environment-variables/#snap-common) and [$SNAP_DATA](https://snapcraft.io/docs/reference/development/environment-variables/#snap-data) directories. The Livepatch client snap ensures that only the root user has read and write access to the sensitive data stored in these directories. It is recommended to maintain strict privileged access controls, to prevent unauthorized access of the snap data.

## Related Security Documentation

[Patch Security](/livepatch/reference/patch-security.md): This document provides in-depth details on the cryptographic approaches used by the Livepatch client for patch verification, signatures, and secure communication.

[Secure Client Operation](/livepatch/how-to-guides/securely-configure-and-operate-the-client.md): This document provides information on how to securely configure and operate the Livepatch client.

[Secure Client Decommissioning](/livepatch/how-to-guides/securely-decommission-the-client.md): This document provides information on how to securely decommission the Livepatch client snap and its data.

[Security Lifecycle](/livepatch/explanation/security-lifecycle.md):This document provides insights into how the security of the Livepatch client is maintained. It also provides guidelines on how security updates to the Livepatch client snap can be controlled.

[Reporting Security Vulnerabilities](/livepatch/how-to-guides/report-client-vulnerability.md): This document lays out the process to report security vulnerabilities found in the Livepatch client and provides information about the Ubuntu Security disclosure and embargo policy.
