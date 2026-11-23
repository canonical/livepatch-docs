---
myst:
  html_meta:
    description: "Reference for Livepatch Server configuration, logging schema, resource requirements, and network access rules."
---

(server-reference-platform)=

# Platform

Configuration options, logging schema, resource requirements, and network access rules for the Livepatch on-premises server.

## In this section

- [Configuration](/server/reference/platform/configuration.md) — All available configuration options for the Livepatch Server, including Juju and snap deployment methods.
- [Logging schema](/server/reference/platform/logging-schema.md) - Structured application fields, security-event identifiers, and optional request context.
- [Resource requirements](/server/reference/platform/resource-requirements.md) — Minimum CPU, memory, and disk requirements for server components.
- [Network access](/server/reference/platform/network-access.md) — Callers, destinations, and ports for configured frontends, upstreams, databases, storage, and monitoring.

```{toctree}
:titlesonly:
:maxdepth: 1
:hidden:

Configuration <configuration.md>
Logging schema <logging-schema.md>
Resource requirements <resource-requirements.md>
Network access <network-access.md>
```