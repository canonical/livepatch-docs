---
myst:
  html_meta:
    description: "Use S3 for patch storage - learn about this topic in Livepatch on-prem."
---

(server-reference-livepatch-on-prem-with-aws-s3-patch-storage)=

# Livepatch on-prem with AWS S3 patch storage

In an AWS EC2 or Kubernetes deployment of Livepatch on-prem, AWS S3 (or an S3-compatible service such as MinIO or Ceph) provides scalable and durable patch storage.

To configure this, follow these steps:

- Create an S3 bucket in your preferred region (ideally the same region as your Livepatch deployment). Ensure the bucket is not publicly writable.
- Choose an authentication mechanism for the Livepatch Server.
- Grant the required IAM permissions to the server credentials.
- Configure the relevant S3 [config options](/server/reference/platform/configuration.md).

Once configured, Livepatch will store and retrieve patch files from the S3 bucket.

## Supported authentication mechanisms

Livepatch supports multiple authentication methods for AWS S3 and S3-compatible storage:

1. **Ambient IAM credentials (recommended for AWS):** When running on AWS EC2 or EKS, Livepatch automatically retrieves credentials from the ambient environment (EC2 instance metadata or IAM Roles for Service Accounts - IRSA). Leave `patch-storage.s3-access-key` and `patch-storage.s3-secret-key` blank.
2. **Static IAM access keys:** Set `patch-storage.s3-access-key` and `patch-storage.s3-secret-key` explicitly.
3. **STS AssumeRole:** Configure `patch-storage.s3-assume-role-arn` to have Livepatch assume a dedicated IAM role using its primary credentials. You can configure `patch-storage.s3-assume-role-duration` (defaults to 15 minutes) to specify the session lifetime. Note that the target IAM role's `MaxSessionDuration` must be configured at or above this value (AWS default is 1 hour).
4. **S3-compatible storage (MinIO, Ceph):** Set `patch-storage.s3-endpoint` to the service URL, set `patch-storage.s3-use-path-style=true`, and toggle `patch-storage.s3-secure` depending on TLS support.

## Required IAM permissions

The credentials used by the Livepatch Server require permissions for two operational roles:

### Standard server operations

For synchronising patches from upstream and serving them to clients, the credentials must have:

- `s3:GetObject` on `arn:aws:s3:::<bucket>/*`
- `s3:PutObject` on `arn:aws:s3:::<bucket>/*`
- `s3:DeleteObject` on `arn:aws:s3:::<bucket>/*`
- `s3:ListBucket` on `arn:aws:s3:::<bucket>`

### Temporary upload credentials for the patch uploader

When an administrator runs `livepatch-admin patch upload`, the Livepatch Server generates temporary AWS SigV4 Presigned POST policies scoped to the `livepatch-*` object prefix.

To generate valid upload grants:

- The credentials must have `s3:PutObject` on `arn:aws:s3:::<bucket>/livepatch-*` (or bucket-wide).
- If using `patch-storage.s3-assume-role-arn`, the primary credentials must have `sts:AssumeRole` on the target role ARN.
- Upload credential lifetimes are bounded by `patch-storage.upload-credential-ttl` (default 15 minutes) or the remaining STS session lifetime.

## Direct patch delivery from S3

To serve patches directly to clients from S3 rather than proxying bytes through the Livepatch Server, set your server's [URL template config](/server/reference/platform/configuration.md) to:

```
https://<bucket>.s3.<region>.amazonaws.com/{filename}
```

 