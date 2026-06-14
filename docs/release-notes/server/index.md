---
myst:
  html_meta:
    description: "Release Notes - technical reference for Livepatch server."
---


(release-notes-server-release-notes-for-the-canonical-livepatch-server-k8s-charm)=

# Release Notes for the Canonical Livepatch Server (K8s charm)

The [Livepatch Server K8s charm](https://charmhub.io/canonical-livepatch-server-k8s) is the easiest and the recommended way to deploy Livepatch Server on K8s. This charm configures and runs the Livepatch Server, which serves livepatches and associated metadata to clients. Use the latest/stable channel charm for production environments.

## Releases

<details><summary>v1.20.0</summary>
What's New:

- The server now returns excluded CVEs grouped by LSN ID when client configuration settings block patches. This is done with the help of the CVE service which now exposes LSN information.
- Database query optimizations for listing patches.
- A more robust set of input validation rules for APIs

</details>
<details><summary>v1.18.1</summary>
What's New:

- Scripts for migrating Livepatch server configuration from the old format used by the reactive machine charms to the new format.
- Patch ping data can now be configured to write into a different Influx bucket. A new configuration value `PingBucket`, determines the bucket for ping data. If left blank, the pings are sent to bucket based on the old `Bucket` value for backwards compatibility. This helps to maintain separate retention policies for Ping data and server KPI metrics.

Bug Fixes:

- Fixed race condition in PostgreSQL patch storage.

</details>
<details><summary>v1.17.17</summary>
What's New:

- Facilitate UX improvements for patch-delay and cutoff-date features.

Bug Fixes:

- Incorrect increment of the Patch Sync progress indicator, when patches are skipped due to errors, is fixed.

</details>
<details><summary>v1.17.12</summary>
What’s New:

- Added caching support for CVE endpoints using hashes and `ETag` and `If-None-Match` headers.
- Timeout for the CVE sync client is now configurable.
- Security events (authentication, authorization, user and system related) are logged by the Livepatch server.

</details>
