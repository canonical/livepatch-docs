---
myst:
  html_meta:
    description: "Use OpenStack Swift for patch storage - learn about this topic in Livepatch on-prem."
---

(server-reference-livepatch-on-prem-with-swift-patch-storage)=

# Livepatch on-prem with OpenStack Swift patch storage

In OpenStack cloud environments or deployments with existing Swift object storage, OpenStack Swift provides scalable, multi-unit patch storage for Livepatch on-prem.

To configure this, follow these steps:

- Create a Swift container for Livepatch patches. Ensure the container is not publicly readable or writable.
- Configure Keystone credentials for the Livepatch service account.
- Provision a TempURL key on the Swift account or container if using the patch uploader.
- Configure the relevant Swift [config options](/server/reference/platform/configuration.md).

Once configured, Livepatch will store and retrieve patch files from the Swift container.

## Supported authentication mechanisms

Livepatch authenticates to OpenStack Swift using OpenStack Identity (Keystone v3):

- `patch-storage.swift-auth-url`: The Keystone authentication URL (for example, `https://keystone.example.com:5000/v3`).
- `patch-storage.swift-username`: Username of the service account.
- `patch-storage.swift-api-key`: User password or API key.
- `patch-storage.swift-domain`: Keystone domain (for example, `default`).
- `patch-storage.swift-tenant`: OpenStack project/tenant name.
- `patch-storage.swift-container`: Target container name.
- `patch-storage.swift-region`: OpenStack region (optional).

## Required permissions and TempURL configuration

Configure permissions for standard operations and offline patch uploads:

### Standard server operations

The Keystone user must have read, write, and delete permissions on the designated container (typically by assigning the `member` or `admin` role in the project).

### Temporary upload credentials for the patch uploader

When an administrator runs `livepatch-admin patch upload`, the Livepatch Server issues temporary, prefix-scoped Swift TempURLs (`temp_url_prefix=livepatch-`) that allow direct uploads to the container without exposing the server's long-lived credentials.

To generate valid TempURL signatures:

1. Provision a TempURL key on the Swift account or container out-of-band using the OpenStack Swift CLI or API:

   ```bash
   # Set at the container level:
   swift post <container-name> --meta "Temp-URL-Key:<your-secret-key>"

   # Or at the account level:
   swift post --meta "Temp-URL-Key:<your-secret-key>"
   ```

2. Record the identical secret key in the Livepatch Server configuration:

   ```bash
   # Juju charm:
   juju config livepatch patch-storage.swift-temp-url-key="<your-secret-key>"

   # Snap:
   sudo snap set canonical-livepatch-server lp.patch-storage.swift-temp-url-key="<your-secret-key>"
   ```

```{important}
OpenStack Swift does not allow retrieving existing TempURL keys via its API for security reasons. You must set the key out-of-band and configure the exact matching value in `patch-storage.swift-temp-url-key`.
```

TempURL validity duration is controlled by `patch-storage.upload-credential-ttl` (defaults to 15 minutes).
