---
myst:
  html_meta:
    description: "Use GCS for patch storage - learn about this topic in Livepatch on-prem."
---

 
(server-reference-livepatch-on-prem-with-gcs-patch-storage)=

# Livepatch on-prem with Google Cloud Storage patch storage

In a GCP deployment of Livepatch on-prem, Google Cloud Storage (GCS) is a good choice for patch storage if the expected number of client machines is high (over 2000).

To configure this, follow these steps:

- Create a GCS bucket in the preferred region (best if the region is the same as the deployment's). Care needs to be taken to make the bucket not publicly writable, as this would pose a significant security risk.
- Create a service account with permissions to read and write objects in that bucket, or rely on the credentials already bound to the VM (e.g. the GCE metadata service, or workload identity) if it is granted the required permissions.
- Configure the relevant GCS [config options](/server/reference/platform/configuration.md).

Once this is configured, Livepatch will store and retrieve patch files from the GCS bucket.

`patch-storage.gcs-credentials-file` and `patch-storage.gcs-credentials-json` are both optional; when both are omitted, Application Default Credentials are used instead. `patch-storage.gcs-impersonate-service-account` is also optional; when set, the resolved credentials are used to impersonate that service account instead of being used directly.
