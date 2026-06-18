---
myst:
  html_meta:
    description: "Reference listing the data transmitted by Livepatch on-premises deployments to Canonical during periodic patch synchronisation checks."
---

(server-reference-data-sent-to-canonical-servers)=

# Data sent to Canonical servers

Livepatch on-prem deployments periodically send requests to servers hosted by Canonical to check for the availability of new live kernel patches. These requests include the following information:

- Unique ID of the deployment
- Number of machines served by the deployment

Additional machine-level reporting can be enabled by setting the `patch-sync.send-machine-reports` configuration option. For example, using Juju:

```
juju config livepatch patch-sync.send-machine-reports=true
```

Enabling this setting allows Livepatch on-prem to send reports about the exact state of each machine served by the deployment.