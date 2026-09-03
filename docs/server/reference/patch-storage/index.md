---
myst:
  html_meta:
    description: "Patch-Storage - learn about this topic in Livepatch on-prem."
---


(server-reference-patch-storage)=

# Livepatch on-prem patch storage

Livepatch Server supports several different drivers for storing patch files downloaded from livepatch.canonical.com:

1. Local filesystem
2. Swift
3. S3 (and compatible implementations, e.g. MinIO)
4. PostgreSQL
5. Google Cloud Storage (GCS)
6. Azure Blob Storage
7. IBM Cloud Object Storage
8. Oracle Cloud Infrastructure (OCI) Object Storage

The filesystem patch store is easiest to deploy and suits most configurations. However, if there is a need to scale out the Livepatch Server such as have multiple Livepatch Servers running to handle the load, the filesystem patch store should not be used.

In case there is a need to scale out Livepatch on-prem, use one of the object storage backends (s3, gcs, azure, ibm, oracle), postgresql, or swift patch stores. Any patch store should have enough space for storing live kernel patches - currently at least 45GB for all patches, see [this guide](/server/reference/patch-management/patch-sync-filters.md) to filter patches sent to your on-prem instance to specific kernel variants/architectures and lower this requirement.

See the [patch storage](/server/reference/platform/configuration.md) config for all available parameters.

```{toctree}
:titlesonly:
:maxdepth: 2
:glob:
:hidden:

Use S3 for patch storage <use-s3-for-patch-storage.md>
Use GCS for patch storage <use-gcs-for-patch-storage.md>
Use Azure for patch storage <use-azure-for-patch-storage.md>
Use IBM COS for patch storage <use-ibm-for-patch-storage.md>
Use OCI Object Storage for patch storage <use-oracle-for-patch-storage.md>
Migrate patches between storage backends <migrating-patch-storage.md>
```
