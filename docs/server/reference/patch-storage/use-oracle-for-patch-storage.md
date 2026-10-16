---
myst:
  html_meta:
    description: "Use OCI Object Storage for patch storage - learn about this topic in Livepatch on-prem."
---

(server-reference-livepatch-on-prem-with-oracle-patch-storage)=

# Livepatch on-prem with Oracle Cloud Infrastructure Object Storage patch storage

In Oracle Cloud Infrastructure (OCI) deployments, OCI Object Storage provides durable and scalable storage for Livepatch on-prem patches.

To configure this, follow these steps:

- Create an Object Storage bucket in your preferred region and tenancy (ideally the same region as the Livepatch deployment). Ensure the bucket access tier is private.
- Choose an authentication method for the Livepatch Server.
- Grant the required OCI IAM policies to the instance dynamic group or user group.
- Configure the relevant OCI [config options](/server/reference/platform/configuration.md).

Once configured, Livepatch will store and retrieve patch files from the OCI Object Storage bucket.

## Supported authentication mechanisms

Livepatch supports two authentication methods for OCI Object Storage:

1. **Instance principal authentication (recommended for OCI):** When running on an OCI compute instance, omit `patch-storage.oracle-config-file`. Livepatch automatically uses the ambient instance principal identity bound to the compute instance.
2. **OCI configuration file:** Set `patch-storage.oracle-config-file` to point to a valid OCI SDK/CLI configuration file containing user credentials, key path, tenancy OCID, and fingerprint. You can optionally specify `patch-storage.oracle-profile` to select a non-default profile within that file.

## Required OCI IAM policies

Configure OCI IAM policies for your instance dynamic group (when using instance principals) or user group (when using a config file):

### Standard server operations

For synchronising patches and delivering them to clients, the identity requires object read, write, and delete permissions. Add a policy statement such as:

```text
allow dynamic-group <livepatch-instance-group> to manage objects in compartment <compartment-name> where target.bucket.name = '<bucket-name>'
```

```{note}
Admin-tool-facilitated patch upload (`livepatch-admin patch upload`) is not supported for OCI Object Storage just yet.
```

