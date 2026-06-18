---
myst:
  html_meta:
    description: "Understand content caching in the Livepatch client, including how SHA256 checksums and HTTP headers reduce network bandwidth when retrieving CVE data."
---

(client-reference-content-caching)=

# Content caching

The Livepatch client uses content caching to reduce network bandwidth when retrieving CVE data from the Livepatch server. This document explains where and how content caching with hashes is applied.

## Where content caching is used

The Livepatch client retrieves CVE data from the hosted Livepatch server for the kernel packages present on the machine. If a newer version of the kernel fixes a patched CVE, the client can display information about the CVEs fixed in the new kernel version. Content caching is used when retrieving this information.

## Why content caching is required

CVE data for a machine changes infrequently — only an update to the CVE data source or the installation of a new kernel package triggers a change. Sending this data to the client on every request is unnecessary.

## How content caching works

The hosted Livepatch server and the Livepatch client rely on content hashes to send up-to-date CVE data efficiently:

- An SHA256 checksum tracks updates to the CVE data.
- The client sends its stored SHA256 checksum to the server using the `If-None-Match` HTTP header.
- If the checksums differ, the server sends the updated CVE data along with the new SHA256 checksum using the `ETag` header.
- The client stores this checksum with the content for use in future requests.
