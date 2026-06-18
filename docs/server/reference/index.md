---
myst:
  html_meta:
    description: "Technical reference for the Livepatch on-premises server, covering platform configuration, authentication, patch management, patch storage, telemetry, and release notes."
---

(server-reference)=

# Reference

This section provides technical reference information for the Livepatch on-premises server.

## In this section

- [Platform](/server/reference/platform/index.md) — Configuration options, resource requirements, and network access rules.
- [Authentication](/server/reference/authentication/index.md) — Authentication and authorization mechanisms for the Livepatch Server APIs.
- [Patch management](/server/reference/patch-management/index.md) — Managing patch tiers, promotion workflows, and sync filter options.
- [Patch storage](/server/reference/patch-storage/index.md) — Configuring patch storage backends such as S3 for the Livepatch Server.
- [Telemetry](/server/reference/telemetry/index.md) — Data sent to Canonical during patch synchronisation and machine reporting.
- [Releases](/release-notes/server/index.md) — Release notes for the Livepatch Server.

```{toctree}
:titlesonly:
:maxdepth: 2
:hidden:

Platform <platform/index.md>
Authentication <authentication/index.md>
Patch management <patch-management/index.md>
Patch storage <patch-storage/index.md>
Telemetry <telemetry/index.md>
Releases <../../release-notes/server/index.md>
```