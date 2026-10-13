---
myst:
  html_meta:
    description: "Use Azure for patch storage - learn about this topic in Livepatch on-prem."
---

(server-reference-livepatch-on-prem-with-azure-patch-storage)=

# Livepatch on-prem with Azure Blob Storage patch storage

In Microsoft Azure deployments, Azure Blob Storage offers resilient and scalable patch storage for Livepatch on-prem.

To configure this, follow these steps:

- Create a Blob Storage container in your preferred region (ideally the same region as the Livepatch deployment). Ensure the container public access level is set to private.
- Choose an authentication method for the Livepatch Server.
- Assign the required Azure role-based access control (RBAC) roles.
- Configure the relevant Azure [config options](/server/reference/platform/configuration.md).

Once configured, Livepatch will store and retrieve patch files from the Azure Blob Storage container.

## Supported authentication mechanisms

Livepatch supports several authentication methods for Azure Blob Storage:

1. **Storage account key:** Set `patch-storage.azure-account-name` and `patch-storage.azure-account-key`.
2. **Storage connection string:** Set `patch-storage.azure-connection-string`. This option supersedes individual account name and key settings.
3. **Microsoft Entra ID service principal:** Configure `patch-storage.azure-tenant-id`, `patch-storage.azure-client-id`, and `patch-storage.azure-client-secret`.
4. **System-assigned managed identity (recommended for Azure):** When running on an Azure Virtual Machine or Azure Kubernetes Service (AKS), omit keys, connection strings, and service principal secrets. Livepatch automatically uses the ambient system-assigned managed identity.
5. **User-assigned managed identity:** When using a user-assigned managed identity instead of the system-assigned default, specify its client ID with `patch-storage.azure-managed-identity-client-id`.

## Required Azure RBAC roles

Depending on the chosen authentication method, assign the appropriate Azure RBAC roles to the Livepatch identity:

### Standard server operations

For synchronising and serving patches, the identity must have blob read, write, and delete permissions. Assign the following role at the storage account or container scope:

- **Storage Blob Data Contributor**

### Temporary upload credentials for the patch uploader

When an administrator runs `livepatch-admin patch upload`, the Livepatch Server generates container-level Shared Access Signature (SAS) tokens granting write and create capabilities:

- **When using Account Key or Connection String:** Livepatch signs the SAS token directly using the storage account Shared Key.
- **When using Microsoft Entra ID (Service Principal or Managed Identity):** Livepatch requests a User Delegation Key from Microsoft Entra ID to sign a User Delegation SAS. To enable this, assign:
  1. **Storage Blob Delegator** (or custom role with `Microsoft.Storage/storageAccounts/blobServices/generateUserDelegationKey/action`) scoped at the storage account level.
  2. **Storage Blob Data Contributor** scoped at the storage account or target container level.

Upload credential lifetimes are governed by `patch-storage.upload-credential-ttl` (defaults to 15 minutes).

