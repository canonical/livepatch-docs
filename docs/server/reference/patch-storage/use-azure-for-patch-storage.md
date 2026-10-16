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

```{note}
**`patch-storage.azure-account-name` is required in all authentication modes except when using a connection string** (which already carries the account name). This includes managed identity and service principal modes.
```

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

When an administrator runs `livepatch-admin patch upload`, the Livepatch Server generates container-level Shared Access Signature (SAS) tokens granting write and create capabilities throughout the container:

- **When using Account Key or Connection String:** Livepatch signs the SAS token directly using the storage account Shared Key.
- **When using Microsoft Entra ID (Service Principal or Managed Identity):** Livepatch requests a User Delegation Key from Microsoft Entra ID to sign a User Delegation SAS. To enable this, assign:
  1. **Storage Blob Delegator** (or custom role with `Microsoft.Storage/storageAccounts/blobServices/generateUserDelegationKey/action`) scoped at the storage account level.
  2. **Storage Blob Data Contributor** scoped at the storage account or target container level.

Upload credential lifetimes are governed by `patch-storage.upload-credential-ttl` (defaults to 15 minutes).

The `Storage Blob Delegator` role grants the `Microsoft.Storage/storageAccounts/blobServices/generateUserDelegationKey/action` permission, which the server needs to sign User Delegation SAS tokens when authenticating via Microsoft Entra ID. Without it, `livepatch-admin patch upload` fails with a 403 when using managed identity or service principal auth.

### Assigning roles per authentication mechanism

The identity that must hold the roles above depends on the authentication mechanism:

| Auth mechanism | Identity to assign roles to |
|----------------|-----------------------------|
| Account key / connection string | None (shared-key auth — no RBAC needed) |
| Service principal | The service principal's app ID |
| System-assigned managed identity | The VM/VMSS system-assigned principal ID |
| User-assigned managed identity | The user-assigned identity's principal ID |

