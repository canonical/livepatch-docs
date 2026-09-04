---
myst:
  html_meta:
    description: "Release notes for the Canonical Livepatch Server K8s charm. Find new features, bug fixes, and changes for each server release."
---

(release-notes-server)=

# Livepatch Server release notes

The [Livepatch Server K8s charm](https://charmhub.io/canonical-livepatch-server-k8s) is the recommended method for deploying the Livepatch Server on Kubernetes. The charm configures and runs the Livepatch Server, which serves live kernel patches and associated metadata to clients. Use the `latest/stable` channel charm for production environments.

## v2.3.0

### New features

- Added support for Public Cloud Stores (AWS S3, Azure Blob Store, Google Cloud Storage, IBM COS, Oracle Cloud Storage) to be used as patch stores.

### Optimizations

- Mark the comparable_patch_version function as parallel safe for faster patch version lookups.

### Deprecations

- Deprecate SSO Macaroon issuance, SSO Macaroon based admin auth and SSO Auth config from the server.

## v2.0.0

### New Features

- Removed the `/api/auth-tokens` endpoint. 
- Ping metrics can now be pushed directly to an OTLP collector.
- Traces can now be pushed directly to an OTLP collector.

### Bug Fixes

- Updated dependencies to pull in upstream CVE fixes.

## v1.21.3

### New features

- Added support for new ping types sent by client machines which improves monitoring of client machines. 
- Fixed CVE data is now denormalized for faster database lookups. 

### Bug Fixes:

- Fixed config parsing logic which caused invalid handling of nested config struct pointers. 


## v1.20.0

### New features

- Excluded CVEs are now returned grouped by LSN ID when client configuration settings block patches. This integration uses the CVE service, which now exposes LSN information.
- Database query optimizations for listing patches.
- More robust input validation rules for API endpoints.

## v1.18.1

### New features

- Scripts for migrating Livepatch Server configuration from the old reactive machine charm format to the new format.
- A new configuration value, `PingBucket`, determines the Influx bucket for patch ping data. If left blank, pings are sent to the bucket specified by the existing `Bucket` value for backwards compatibility. This enables separate retention policies for ping data and server KPI metrics.

### Bug fixes

- Fixed a race condition in PostgreSQL patch storage.

## v1.17.17

### New features

- UX improvements for the patch-delay and cutoff-date features.

### Bug fixes

- Fixed an issue where the patch sync progress indicator incremented incorrectly when patches were skipped due to errors.

## v1.17.12

### New features

- Added caching support for CVE endpoints using hashes and `ETag` and `If-None-Match` headers.
- The timeout for the CVE sync client is now configurable.
- Security events (authentication, authorization, user and system related) are now logged by the Livepatch Server.
