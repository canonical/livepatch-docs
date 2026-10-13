---
myst:
  html_meta:
    description: "Use GCS for patch storage - learn about this topic in Livepatch on-prem."
---

(server-reference-livepatch-on-prem-with-gcs-patch-storage)=

# Livepatch on-prem with Google Cloud Storage patch storage

In Google Cloud Platform (GCP) deployments, Google Cloud Storage (GCS) provides highly available and scalable patch storage for Livepatch on-prem.

To configure this, follow these steps:

- Create a GCS bucket in your preferred region (ideally the same region as the Livepatch deployment). Ensure the bucket is not publicly writable.
- Choose an authentication mechanism for the Livepatch Server service account.
- Grant the required IAM roles to the service account on the bucket.
- Configure the relevant GCS [config options](/server/reference/platform/configuration.md).

Once configured, Livepatch will store and retrieve patch files from the GCS bucket.

## Supported authentication mechanisms

Livepatch supports several authentication methods for Google Cloud Storage:

1. **Application Default Credentials (ADC, recommended for GCP):** When running on Google Compute Engine (GCE) or Google Kubernetes Engine (GKE) with Workload Identity, Livepatch automatically acquires credentials from the ambient environment. Leave `patch-storage.gcs-credentials-file` and `patch-storage.gcs-credentials-json` unset.
2. **Service account key file:** Specify the absolute path to a service account JSON key file via `patch-storage.gcs-credentials-file`.
3. **Inline service account JSON:** Provide the raw JSON key contents directly using `patch-storage.gcs-credentials-json`.
4. **Service account impersonation:** Set `patch-storage.gcs-impersonate-service-account` to the email of a target service account. Livepatch uses the primary resolved credentials to impersonate that account.

## Required IAM permissions

The service account used by the Livepatch Server requires permissions for standard operations and offline patch uploads:

### Standard server operations

For synchronising patches and serving them to clients, the service account must have read, write, and delete permissions on the bucket. Assigning the predefined role `roles/storage.objectUser` (or `roles/storage.objectAdmin`) at the bucket level satisfies these requirements:

- `storage.objects.get`
- `storage.objects.create`
- `storage.objects.delete`
- `storage.objects.list`

### Temporary upload credentials for the patch uploader

When an administrator runs `livepatch-admin patch upload`, the Livepatch Server uses the Google Cloud Security Token Service (STS) to generate downscoped OAuth 2.0 bearer access tokens. These tokens are restricted via a Credential Access Boundary (CAB) to objects beginning with the `livepatch-` prefix within the bucket.

To issue and accept these downscoped upload tokens:

- The service account must hold `roles/storage.objectUser` on the bucket (both `storage.objects.create` and `storage.objects.delete` are required for overwrite support).
- The service account must have permission to perform token exchanges with Google Cloud STS.

```{note}
**Upload token lifetime:** Downscoped access tokens issued by Google Cloud STS have a fixed duration of ~1 hour (3600 seconds) enforced by Google Cloud. Configuring `patch-storage.upload-credential-ttl` does not shorten or extend this cloud-enforced token lifetime.
```

