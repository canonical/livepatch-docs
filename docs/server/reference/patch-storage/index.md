---
myst:
  html_meta:
    description: "Patch-Storage - learn about this topic in Livepatch on-prem."
---


(server-reference-patch-storage)=

# Livepatch on-prem patch storage

Livepatch Server supports several different drivers for storing patch files downloaded from livepatch.canonical.com or uploaded across an air gap:

1. **Local filesystem**: Stores patch files directly on disk. Ideal for single-unit deployments.
2. **PostgreSQL**: Stores patch files as Large Objects directly inside the PostgreSQL database. Suitable for multi-unit clusters without external cloud dependencies.
3. **AWS S3**: Highly scalable object storage for AWS or S3-compatible environments (such as MinIO or Ceph).
4. **Google Cloud Storage (GCS)**: Scalable object storage for Google Cloud Platform.
5. **Azure Blob Storage**: Durable container storage for Microsoft Azure.
6. **IBM Cloud Object Storage (COS)**: Resilient bucket storage for IBM Cloud.
7. **Oracle Cloud Infrastructure (OCI) Object Storage**: High-performance object storage for OCI.
8. **OpenStack Swift**: Scalable object storage for OpenStack environments.

The filesystem patch store is easiest to deploy and suits single-unit configurations. However, when scaling out the Livepatch Server across multiple load-balanced units, use PostgreSQL or one of the object storage backends (S3, GCS, Azure, IBM COS, OCI, Swift).

Any patch store should have enough capacity for storing kernel live patches—currently at least 45GB for the full catalog. See [patch sync filters](/server/reference/patch-management/patch-sync-filters.md) to restrict downloads to specific kernel flavours or architectures and reduce storage requirements.

## Authentication and credential models

Livepatch Server supports native, modern authentication mechanisms for each cloud and enterprise backend:

| Storage Backend | Authentication Mechanisms |
| --------------- | ------------------------- |
| **Local filesystem** | Local directory path permissions (`patch-storage.filesystem-path`). |
| **PostgreSQL** | Database connection string (`patch-storage.postgres-connection-string`). |
| **AWS S3** | Ambient IAM credentials (EC2 instance profile, IRSA), static access keys (`patch-storage.s3-access-key`, `patch-storage.s3-secret-key`), or STS AssumeRole (`patch-storage.s3-assume-role-arn`). |
| **Google Cloud Storage** | Application Default Credentials (ADC: GCE metadata, GKE Workload Identity), Service Account key file (`patch-storage.gcs-credentials-file`), inline JSON (`patch-storage.gcs-credentials-json`), or Service Account Impersonation (`patch-storage.gcs-impersonate-service-account`). |
| **Azure Blob Storage** | Storage Account Key (`patch-storage.azure-account-key`), Connection String (`patch-storage.azure-connection-string`), Entra ID Service Principal (`patch-storage.azure-tenant-id`, `patch-storage.azure-client-id`, `patch-storage.azure-client-secret`), or Managed Identity (`patch-storage.azure-managed-identity-client-id`). |
| **IBM Cloud Object Storage** | IAM API key with service instance ID (`patch-storage.ibm-api-key`, `patch-storage.ibm-service-instance-id`), or ambient VPC Instance Metadata Service with an optional trusted profile (`patch-storage.ibm-trusted-profile-id`). |
| **OCI Object Storage** | Instance principal authentication (default on OCI compute), or OCI SDK configuration file and profile (`patch-storage.oracle-config-file`, `patch-storage.oracle-profile`). |
| **OpenStack Swift** | Keystone v3 username, API key/password, domain, tenant, and auth URL (`patch-storage.swift-*`). |

## Upload credentials for the patch uploader

When administrators use `livepatch-admin patch upload` to distribute patches to an on-premises server across an air gap, the server authorizes uploads depending on the storage driver:

- **Cloud object storage (S3, GCS, Azure, IBM COS, OCI, Swift):** The Livepatch Server acts as a credential issuer, generating temporary, scoped, upload-only credentials or presigned URLs. The `livepatch-admin` CLI uploads binary archives directly to cloud storage, bypassing the Livepatch Server API for high-bandwidth file transfers.
- **Server-mediated storage (Local Filesystem, PostgreSQL):** The CLI streams files directly to the Livepatch Server API endpoint (`PUT /v1/admin/patches/file/{filename}`) authenticated by the administrator's login token.
- **Upload credential lifetime:** For cloud backends, the duration of temporary upload credentials is configurable via `patch-storage.upload-credential-ttl` (defaults to 15 minutes). Note that Google Cloud STS (GCS) and IBM Cloud IAM issue tokens with a cloud-enforced fixed duration of ~1 hour.

See the [platform configuration reference](/server/reference/platform/configuration.md) for all available parameters.

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
Use Swift for patch storage <use-swift-for-patch-storage.md>
Migrate patches between storage backends <migrating-patch-storage.md>
```
