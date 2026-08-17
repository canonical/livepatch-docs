---
myst:
  html_meta:
    description: "Use Azure for patch storage - learn about this topic in Livepatch on-prem."
---

 
(server-reference-livepatch-on-prem-with-azure-patch-storage)=

# Livepatch on-prem with Azure Blob Storage patch storage

In an Azure deployment of Livepatch on-prem, Azure Blob Storage is a good choice for patch storage if the expected number of client machines is high (over 2000).

To configure this, follow these steps:

- Create a Blob Storage container in the preferred region (best if the region is the same as the deployment's). Care needs to be taken to make the container not publicly writable, as this would pose a significant security risk.
- Choose an authentication method: a storage account name/key pair, a connection string, an Entra ID (Azure AD) service principal, or a managed identity bound to the VM.
- Configure the relevant Azure [config options](/server/reference/platform/configuration.md).

Once this is configured, Livepatch will store and retrieve patch files from the Azure Blob Storage container.

`patch-storage.azure-account-name` is not required when `patch-storage.azure-connection-string` is set, since the connection string already carries the account name and key. When account key, connection string, and service principal credentials are all omitted, a managed identity bound to the VM is used instead (the system-assigned identity by default, or a user-assigned identity selected via `patch-storage.azure-managed-identity-client-id`).
