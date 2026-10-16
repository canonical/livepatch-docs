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
3. **STS AssumeRole:** Configure `patch-storage.s3-assume-role-arn` to have Livepatch assume a dedicated IAM role using its primary credentials. You can configure `patch-storage.s3-assume-role-duration` (defaults to 15 minutes) to specify the session lifetime. Note that the target IAM role's `MaxSessionDuration` must be configured at or above this value (AWS default is 1 hour), and that values below 15 minutes are rejected by AWS STS (900 seconds is AWS's minimum for `AssumeRole`). The target role's trust policy must also allow the primary identity to assume it (see [Required IAM permissions](#required-iam-permissions)).
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

### STS AssumeRole trust policy requirement

The primary credentials having `sts:AssumeRole` is necessary but not sufficient. The target role's **trust policy** must also list the primary identity (instance role ARN or IAM user ARN) as an allowed principal; without this, AssumeRole fails with 403 even when the caller has the `sts:AssumeRole` permission attached. For example, when the primary credentials come from an EC2 instance profile role `livepatch-server-role`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::<account-id>:role/livepatch-server-role"},
    "Action": "sts:AssumeRole"
  }]
}
```

### Where to attach the S3 policy

The required S3 actions must be attached to the identity that performs the storage operations, which depends on the authentication mode:

| Auth mode | Where the S3 policy must be attached |
|-----------|--------------------------------------|
| Ambient IAM | The instance profile role (or IRSA role for EKS) |
| Static keys | The IAM user |
| STS AssumeRole | The target role (the primary identity only needs `sts:AssumeRole` + the trust policy) |

### Complete IAM policy document

The following policy covers the server's standard operations and the patch uploader. Replace `<bucket>` with your bucket name:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::<bucket>/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::<bucket>"
    }
  ]
}
```

## Direct patch delivery from S3

 To serve patches directly to clients from S3 rather than proxying bytes through the Livepatch Server, enable `server.redirect_downloads` and set your server's [URL template config](/server/reference/platform/configuration.md) to a client-readable endpoint. This static template cannot sign requests: a private S3 bucket needs an authenticated delivery mechanism, or client downloads will fail with 403. For publicly readable objects, use:

```
https://<bucket>.s3.<region>.amazonaws.com/{filename}
```

 