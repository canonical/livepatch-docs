---
myst:
  html_meta:
    description: "Data sent - learn about this topic in Livepatch on-prem."
---


(livepatch_on_prem-explanation-data-sent-to-canonical-servers)=

# Data sent to Canonical servers

Livepatch on-prem deployments periodically send requests to servers hosted by Canonical to check for the availability of new livepatches. These requests contain a unique ID of the deployment and the number of machines served by the deployment.

- Unique ID of the deployment
- Number of machines served by the deployment

Additional reporting can be enabled by running

```
juju config livepatch sync_send_machine_reports=true
```

Changing this config setting will allow livepatch on-prem to send reports about the exact state of each machine served by the deployment.
