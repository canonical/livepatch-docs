---
myst:
  html_meta:
    description: "Use IBM Cloud Object Storage for patch storage - learn about this topic in Livepatch on-prem."
---

 
(server-reference-livepatch-on-prem-with-ibm-patch-storage)=

# Livepatch on-prem with IBM Cloud Object Storage patch storage

In an IBM Cloud deployment of Livepatch on-prem, IBM Cloud Object Storage (COS) is a good choice for patch storage if the expected number of client machines is high (over 2000).

To configure this, follow these steps:

- Create a COS bucket in the preferred region (best if the region is the same as the deployment's). Care needs to be taken to make the bucket not publicly writable, as this would pose a significant security risk.
- Choose an authentication method: HMAC credentials, an IAM API key with a service instance ID, or the ambient VPC Instance Metadata Service if the VM is granted the required trusted profile.
- Configure the relevant IBM [config options](/server/reference/platform/configuration.md).

Once this is configured, Livepatch will store and retrieve patch files from the COS bucket.

Credentials are resolved in the following order of preference: HMAC keys (`patch-storage.ibm-access-key`/`patch-storage.ibm-secret-key`), then an IAM API key (`patch-storage.ibm-api-key`/`patch-storage.ibm-service-instance-id`), then, if none are configured, the ambient VPC Instance Metadata Service (optionally selecting a trusted profile via `patch-storage.ibm-trusted-profile-id`).
