---
myst:
  html_meta:
    description: "Use S3 for patch storage - learn about this topic in Livepatch on-prem."
---

 
(livepatch_on_prem-explanation-livepatch-on-prem-with-aws-s3-patch-storage)=

# Livepatch on-prem with AWS S3 patch storage

In an AWS EC2 deployment of livepatch on-prem, it makes sense to use S3 for patch storage if the expected number of client machines is high (over 2000).

To configure this, follow these steps:

- Create an S3 bucket in the preferred region (best if the region is the same as the deployment's). Care needs to be taken to make the bucket not world-writable as this would pose a significant security risk.
- Create an access point with permissions to perform operations on that S3 bucket.
- Create a programmatic IAM user account with permissions to perform S3 operations.
- Configure the relevant S3 [config options](/livepatch_on_prem/reference/configuration.md)

Once this is configured, livepatch will store and retrieve patch files from the S3 bucket.

A further improvement is to configure livepatch on-prem to serve patches from the S3 bucket directly. For that public http access needs to be allowed to that bucket. Set your server's [URL template config](/livepatch_on_prem/reference/configuration.md) to something resembling:

```
https://<bucket.s3-<region>.amazonaws.com/{filaname}
```
 