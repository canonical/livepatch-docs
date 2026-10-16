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

1. **Application Default Credentials (ADC, recommended for GCP):** When running on Google Compute Engine (GCE) or Google Kubernetes Engine (GKE) with Workload Identity, Livepatch automatically acquires credentials from the ambient environment. Leave `patch-storage.gcs-credentials-file` and `patch-storage.gcs-credentials-json` unset. With ADC on GCE, the identity used is the service account attached to the VM instance, so grant `roles/storage.objectUser` on the bucket to that attached service account (verify with `gcloud compute instances describe <vm> --format='value(serviceAccounts[0].email)'`).
2. **Service account key file:** Specify the absolute path to a service account JSON key file via `patch-storage.gcs-credentials-file`. When deploying with the snap, the file must live inside the snap's writable area (see the snap note below).
3. **Inline service account JSON:** Provide the raw JSON key contents directly using `patch-storage.gcs-credentials-json`. When deploying with the snap, pass the JSON as a JSON-encoded string (see the snap note below).
4. **Service account impersonation:** Set `patch-storage.gcs-impersonate-service-account` to the email of a target service account. Livepatch uses the primary resolved credentials to impersonate that account. The primary identity must be granted `roles/iam.serviceAccountTokenCreator` on the target service account (see [Required IAM permissions](#required-iam-permissions)).

````{note}
**Snap deployments are strictly confined.** The livepatch server snap cannot read a service account key file at an arbitrary path (for example `/tmp/gcs-key.json` or `/home/ubuntu/...`). Place the key file inside the snap's common directory and reference that path:

```bash
sudo mkdir -p /var/snap/canonical-livepatch-server/common
sudo cp gcs-key.json /var/snap/canonical-livepatch-server/common/gcs-key.json
sudo snap set canonical-livepatch-server \
  lp.patch-storage.gcs-credentials-file=/var/snap/canonical-livepatch-server/common/gcs-key.json
```
````

````{note}
**Snap deployments parse inline JSON.** `snap set` auto-parses JSON-looking values into nested objects, so passing the raw key JSON results in the value being stored as a map rather than a string. Pass the key JSON as a JSON-encoded string:

```bash
sudo snap set canonical-livepatch-server \
  lp.patch-storage.gcs-credentials-json="$(jq -c . gcs-key.json)"
```

Verify with `sudo snap get canonical-livepatch-server lp.patch-storage.gcs-credentials-json` — the value must be a quoted JSON string, not a nested object.
````

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
- The service account must hold `roles/iam.serviceAccountTokenCreator` **on itself** (this grants `iam.serviceAccounts.getAccessToken`, required to obtain the root token via `iamcredentials.generateAccessToken`). The STS downscoping exchange itself requires no additional IAM permission.

```{note}
**Upload token lifetime:** Downscoped access tokens issued by Google Cloud STS have a fixed duration of ~1 hour (3600 seconds) enforced by Google Cloud. Configuring `patch-storage.upload-credential-ttl` does not shorten or extend this cloud-enforced token lifetime.
```

### Service account impersonation

When using service account impersonation (`patch-storage.gcs-impersonate-service-account`), the **target** service account is the identity that actually reads and writes the bucket; the primary (source) identity only obtains tokens for it.

- Grant the primary identity `roles/iam.serviceAccountTokenCreator` on the target service account (the one named in `patch-storage.gcs-impersonate-service-account`):

  ```bash
  gcloud iam service-accounts add-iam-policy-binding \
    <target-sa-email> \
    --member "serviceAccount:<primary-sa-email>" \
    --role roles/iam.serviceAccountTokenCreator
  ```

- Grant `roles/storage.objectUser` on the bucket to the **target** service account, since that is the identity that performs the actual storage operations. The source identity only needs the token-creator permission above.

### Summary of required permissions

The complete permission set for the server's IAM identity (service account) is:

| Purpose | Role / permission | Scope |
|---------|-------------------|-------|
| Sync & serve patches | `roles/storage.objectUser` (`storage.objects.get/create/delete/list`) | Bucket |
| Patch uploader — root token for downscoping | `roles/iam.serviceAccountTokenCreator` (`iam.serviceAccounts.getAccessToken`) | On the service account itself |
| Patch uploader — impersonation (optional mode) | `roles/iam.serviceAccountTokenCreator` | On the impersonation target service account, granted to the primary identity |
| Patch uploader — impersonation target storage access | `roles/storage.objectUser` | Bucket, granted to the impersonation target service account |

There is **no** separate "STS token exchange" permission — that step is permissionless once the root token exists.

