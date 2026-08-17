---
myst:
  html_meta:
    description: "Use OCI Object Storage for patch storage - learn about this topic in Livepatch on-prem."
---

 
(server-reference-livepatch-on-prem-with-oracle-patch-storage)=

# Livepatch on-prem with Oracle Cloud Infrastructure Object Storage patch storage

In an OCI deployment of Livepatch on-prem, Oracle Cloud Infrastructure (OCI) Object Storage is a good choice for patch storage if the expected number of client machines is high (over 2000).

To configure this, follow these steps:

- Create an Object Storage bucket in the preferred region (best if the region is the same as the deployment's). Care needs to be taken to make the bucket not publicly writable, as this would pose a significant security risk.
- Choose an authentication method: an OCI config file/profile, or instance principal authentication if the compute instance is granted the required IAM policies.
- Configure the relevant OCI [config options](/server/reference/platform/configuration.md).

Once this is configured, Livepatch will store and retrieve patch files from the OCI Object Storage bucket.

`patch-storage.oracle-config-file` (and `patch-storage.oracle-profile`) are optional; when omitted, instance principal authentication is used instead, bound to the OCI compute instance the server runs on.
