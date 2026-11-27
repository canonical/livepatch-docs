---
myst:
  html_meta:
    description: "Reference listing the data transmitted by Livepatch on-premises deployments to Canonical during periodic patch synchronisation checks."
---

(server-reference-data-sent-to-canonical-servers)=

# Data sent to Canonical servers

Livepatch on-prem deployments periodically send synchronization requests to the upstream configured by `patch-sync.upstream-url`, using the credential configured through `patch-sync.token` or, for charm deployments, the preferred `patch-sync.token-secret`. For connected deployments, this upstream is Canonical's hosted Livepatch service or another on-premises server. These requests include the following information:

- Unique ID of the deployment
- Machine-count information, according to `patch-sync.machine-count-strategy`
- Whether detailed machine-report forwarding is enabled
- Patch-selection information, such as the requested tier, architectures, kernel flavours, and existing patch inventory

Clients send their requests and reports to your on-premises server, not directly to Canonical. Forwarding of detailed machine reports upstream is disabled by default and is separate from local report storage. Enable `patch-sync.send-machine-reports` only if you intend your configured upstream to receive machine-level information. For example, using Juju:

```
juju config livepatch patch-sync.send-machine-reports=true
```

When enabled, forwarded reports include architecture, CPU model, kernel and Livepatch Client versions, boot time, uptime, last-check time, and patch state and version. Machine IDs are pseudonymized, not anonymized: before forwarding, each is replaced with an HMAC-SHA256 hash computed with a fixed key built into the server. The hash is stable for each machine, so the upstream can correlate reports from the same machine without receiving its raw ID, but anyone who knows a machine ID can compute its hash. Disabling this option does not stop the deployment ID, machine counts, and patch-selection information being sent during upstream synchronization.

## Air-gapped deployments

Your on-premises server can operate without direct communication with Canonical, using local auth and machine tokens and {ref}`offline patch imports <server-how-to-guides-distribute-patches-offline>`. An internet-connected workstation can download patches for transfer into the isolated environment; the air-gapped server does not need to send client reports to that workstation or to Canonical. Keep any internal upstream servers, patch storage, and observability services inside the isolated environment.