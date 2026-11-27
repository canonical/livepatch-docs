---
myst:
  html_meta:
    description: "Reference explaining upstream synchronization data, optional machine-report forwarding, and air-gapped operation for Livepatch on-premises deployments."
---

(server-reference-data-sent-to-canonical-servers)=

# Data sent to Canonical servers

The Livepatch Server communicates with its configured upstream to synchronize patches. For connected deployments using Canonical's hosted Livepatch service as the upstream, synchronization sends the information described below to Canonical. The upstream can instead be another on-premises server, which receives the synchronization requests. Air-gapped operation is described separately below.

## Upstream synchronization

Synchronization requests go to the upstream configured by `patch-sync.upstream-url`, using the credential in `patch-sync.token`. Requests include:

- The deployment ID
- Machine-count information, according to `patch-sync.machine-count-strategy`
- Whether detailed machine-report forwarding is enabled
- Patch-selection information, such as the requested tier, architectures, kernel flavours, and existing patch inventory

## Local reporting and optional forwarding

Clients configured to use your on-premises Livepatch Server send their requests and reports to that server, not directly to Canonical's hosted Livepatch service. With the documented {ref}`auth-token workflow <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`, client registration and machine-token validation take place within your deployment.

This local client traffic is separate from your server's upstream synchronization. Enabling local machine-report storage does not enable forwarding of detailed reports upstream.

Detailed machine-report forwarding is disabled by default. Enable `patch-sync.send-machine-reports` only if you intend your configured upstream to receive machine-level information. For example, using Juju:

```
juju config livepatch patch-sync.send-machine-reports=true
```

When enabled, forwarded reports include architecture, CPU model, kernel and Livepatch Client versions, boot time, uptime, last-check time, and patch state and version. Machine IDs are replaced with an HMAC-SHA256 hash before forwarding. The hash is stable for each machine, so the upstream can correlate reports from the same machine without receiving its raw ID.

Disabling this option prevents detailed machine-report forwarding; it does not stop deployment IDs, machine counts, and patch-selection information being sent during upstream synchronization. Local report storage and upstream report forwarding are separate settings.

## Air-gapped deployments

Your on-premises server can operate without direct communication with Canonical, using local auth and machine tokens and {ref}`offline patch imports <server-how-to-guides-distribute-patches-offline>`. An internet-connected workstation can download patches for transfer into the isolated environment; the air-gapped server does not need to send client reports to that workstation or to Canonical.

Keep any internal upstream servers, patch storage, and observability services inside the isolated environment. Local report retention and access controls still apply.