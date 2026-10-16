---
myst:
  html_meta:
    description: "Use IBM Cloud Object Storage for patch storage - learn about this topic in Livepatch on-prem."
---

(server-reference-livepatch-on-prem-with-ibm-patch-storage)=

# Livepatch on-prem with IBM Cloud Object Storage patch storage

In IBM Cloud deployments, IBM Cloud Object Storage (COS) provides secure and scalable patch storage for Livepatch on-prem.

To configure this, follow these steps:

- Create an IBM Cloud Object Storage bucket in your preferred region (ideally the same region as your deployment). Ensure the bucket is not publicly writable.
- Choose an authentication method for the Livepatch Server.
- Grant the required IAM access policies or trusted profiles.
- Configure the relevant IBM [config options](/server/reference/platform/configuration.md).

Once configured, Livepatch will store and retrieve patch files from the IBM COS bucket.

## Supported authentication mechanisms

Livepatch supports two IAM-based authentication methods for IBM Cloud Object Storage:

1. **IAM API key with service instance ID:** Configure `patch-storage.ibm-api-key` and `patch-storage.ibm-service-instance-id`.
2. **VPC Instance Metadata Service (recommended for IBM Cloud VPC):** When running on an IBM Cloud Virtual Server Instance (VSI), omit the API key. Livepatch automatically queries the ambient VPC Instance Metadata Service (IMS) to authenticate. You can optionally bind the instance to a specific trusted profile via `patch-storage.ibm-trusted-profile-id`.

```{note}
Legacy S3-compatible HMAC credentials (access key and secret key pairs) are unsupported for IBM COS in modern Livepatch releases. IBM Cloud IAM authentication is required.
```

## Required IAM access policies and trusted profiles

Configure IBM Cloud IAM roles for regular patch operations and offline patch uploads:

### Standard server operations

For synchronising and serving patches, the Livepatch Server identity (API key or instance trusted profile) must hold the following roles on the COS bucket or service instance:

- **Reader** (to retrieve objects and list bucket contents)
- **Writer** (or **ObjectWriter**, to write and delete patch files)

```{note}
Admin-tool-facilitated patch upload (`livepatch-admin patch upload`) is not supported for IBM Cloud Object Storage just yet.
```

