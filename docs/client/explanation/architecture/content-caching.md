---
myst:
  html_meta:
    description: "Content caching - technical reference for Livepatch client."
---


(client-reference-content-caching)=

# Content Caching

This document acts as a reference on how and where content caching with hashes is used in the livepatch server, to reduce the network bandwidth required while trying to obtain certain information.

## Where is content caching used in the Livepatch Client?

The livepatch client gets the fixed CVE data from the hosted livepatch server, for the kernel packages currently present in the machine. If a newer version of the kernel fixes a patched CVE, the information of the CVEs fixed in the new kernel version can be viewed using the livepatch client. The Livepatch client uses content caching while retrieving this information from the server.

## Why is content caching required?

The CVE data for a machine changes infrequently, because only an update to the CVE data source or a new kernel package being installed on the machine can trigger a change. Therefore, it does not make sense for the server to send this information to the client on every request.

## How is content caching achieved by the Livepatch client?

The hosted livepatch server and the livepatch client rely on caching using content hashes, to send up-to-date CVE data. An SHA256 checksum is used to track updates to the CVE data. The client sends the stored SHA256 checksum, for the current CVE data, to the server using the `If-None-Match` header. If there is a mismatch between the SHA256 checksums present in the client and the server, the updated CVE data is sent to the client along with the newly computed SHA256 checksum using the `ETag` header. The client stores this SHA256 checksum along with the content for use in future requests.
