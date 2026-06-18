---
myst:
  html_meta:
    description: "Reference for Livepatch on-premises server configuration options, covering server, authentication, database, patch storage, sync, telemetry, and machine reporting settings."
---

(server-reference-configuration)=

# Configuration

This document provides the configuration options for the Livepatch Server for both Juju charm and snap deployments.

```{note}
The configuration below applies to the Livepatch Server operator charms and Server snap.
For reactive charm deployments — or if your deployment configuration differs from the options below — see the [migration guide from reactive charm to operator charm](/server/how-to-guides/deployment/migrate-from-reactive-charm-to-operator-charm.md).
```

## Setting configuration

Configuration methods differ depending on whether the Livepatch Server is deployed with Juju and charms, or as a standalone snap.

Default and example values are available on the respective [machine charm](https://charmhub.io/canonical-livepatch-server/configurations?channel=ops1.x/stable) and [K8s charm](https://charmhub.io/canonical-livepatch-server-k8s/configurations) configuration pages.

### Juju

The configuration keys in the table below map directly to the options exposed by the Livepatch charms, unless otherwise noted.

Assuming the Livepatch Server has been deployed with the alias `livepatch`, apply configuration changes with:

```
juju config livepatch <key>=<value>
# Example: enable basic authentication
juju config livepatch auth.basic.enabled=true
```

See the [Juju CLI configuration documentation](https://documentation.ubuntu.com/juju/latest/reference/juju-cli/list-of-juju-cli-commands/config/) for all available methods of applying configuration.

### Snap

The configuration keys in the table below map directly to the values accepted by the Livepatch Server snap. Prepend `lp.` to all keys when setting snap configuration.

Apply configuration changes with:

```
sudo snap set canonical-livepatch-server lp.<key>=<value>
# Example: enable basic authentication
sudo snap set canonical-livepatch-server lp.auth.basic.enabled=true
```

## Configuration options

The following sections describe the available configuration values.

### Server configuration

| Name | Description | Values |
| ---- | ----------- | ------ |
| `server.log-level` | Log level for the server. | `debug`, `info`, `warn`, `error`, `dpanic`, `panic`, `fatal` |
| `server.redirect_downloads` | Whether to redirect patch download requests to the URL specified in `server.url-template`. | `bool` |
| `server.url-template` | Template URL for redirecting clients for patch downloads. Example: `https://my-file-server.com/{filename}`. | `string` |
| `server.server-address` | Listen address for the server. | `url` |
| `server.concurrency-limit` | Maximum number of API requests to serve concurrently. | `integer` |
| `server.burst-limit` | Queue limit. Approximately equals `concurrency-limit`. | `integer` |
| `server.is-leader` | In multi-server deployments, determines whether this unit is the leader. Not available for charmed deployments. | `bool` |
| `server.is-hosted` | Enable configuration blocks specific to Canonical's hosted Livepatch configuration. | `bool` |

### Admin authentication

The following values configure authentication to the server's admin endpoints. Only Basic Auth and Ubuntu SSO authentication are supported.

- SSO Teams represent Launchpad teams.
- Basic Auth configuration accepts a comma-separated list of users. See the [admin tool setup guide](/server/how-to-guides/security/setup-administration-tool.md#password-authentication) for details.
- Basic Auth passwords must be bcrypt hashed.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `auth.basic.enabled` | Enable Basic Auth. | `bool` |
| `auth.basic.users` | Comma-separated list of user objects. | `<user1>:<bcrypt hashed password>, <user2>:<bcrypt hashed password>` |
| `auth.sso.enabled` | Enable Ubuntu SSO authentication. | `bool` |
| `auth.sso.teams` | SSO authentication configuration listing authorised teams. | `https://launchpad.net/~team-1,https://launchpad.net/~team-2` |
| `auth.sso.url` | URL for SSO authentication. | `login.ubuntu.com` |
| `auth.sso.public-key` | Public key for the authentication server. Accepts a file path or an inline key. | `string` |

### Ubuntu Pro

The following values configure how the server interacts with the Ubuntu Pro backend (also referred to as the contracts server) for client authentication. These are relevant for Canonical's hosted Livepatch server and airgapped deployments.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `contracts.enabled` | Whether to connect to the contracts service. | `bool` |
| `contracts.url` | URL of the contracts server. | `string` |
| `contracts.user` | Basic Auth username for the contracts service. | `string` |
| `contracts.password` | Basic Auth password for the contracts service. | `string` |

### Database

The following values configure how the server interacts with its PostgreSQL database.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `database.connection-string` | PostgreSQL connection string. Unavailable for charmed deployments (handled through Juju relations). | `string` |
| `database.connection-pool-max` | Maximum number of connections in the pool. | `int` |
| `database.connection-lifetime-max` | Maximum lifetime of a database connection. | `int` |
| `database.work-mem` | Maximum memory in MB available for each query operation. | `int` |

### InfluxDB

The following values configure how the server interacts with InfluxDB for sending aggregated KPIs.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `influx.enabled` | Whether to enable InfluxDB KPI reporting (hosted). | `bool` |
| `influx.url` | URL of the InfluxDB server. | `string` |
| `influx.token` | Authentication token for InfluxDB. | `string` |
| `influx.bucket` | InfluxDB bucket to use. | `string` |
| `influx.ping_bucket` | Bucket for sending patch ping data, for example under a different retention policy. If empty, defaults to the value of `influx.bucket`. | `string` |
| `influx.organization` | InfluxDB organisation containing the bucket. | `string` |

### TimescaleDB

The following values configure how the server interacts with its TimescaleDB instance for storing time-series patch ping metrics. TimescaleDB runs as a PostgreSQL extension on a dedicated PostgreSQL instance, separate from the main database. TimescaleDB can be enabled alongside or as a replacement for InfluxDB for KPI metrics.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `timescale-db.enabled` | Whether to enable the TimescaleDB store for KPI metrics. | `bool` |
| `timescale-db.connection-string` | Connection string for the TimescaleDB PostgreSQL instance. | `string` |
| `timescale-db.connection-pool-max` | Maximum number of connections in the pool (1–1000). | `int` |
| `timescale-db.connection-lifetime-max` | Maximum lifetime of a connection before it is recycled. | `duration` |
| `timescale-db.work-mem` | Maximum memory in MB available for each query operation. | `int` |
| `timescale-db.flush-timeout` | Duration the store waits before flushing buffered writes. Defaults to 1 minute if unset. | `duration` |

### Patch storage

The following values configure how the server manages patch storage. See the [patch storage how-to guide](/server/reference/patch-storage/index.md) for details.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `patch-storage.type` | Storage backend type for on-premises patch synchronisation. | `oneof: filesystem, swift, postgres, s3` |
| `patch-storage.filesystem-path` | Directory path for filesystem storage. | `string` |
| `patch-storage.swift-username` | Swift account username. | `string` |
| `patch-storage.swift-api-key` | Swift API key. | `string` |
| `patch-storage.swift-auth-url` | Swift authentication URL. | `string` |
| `patch-storage.swift-domain` | Swift domain. | `string` |
| `patch-storage.swift-tenant` | Swift tenant. | `string` |
| `patch-storage.swift-container` | Swift container bucket. | `string` |
| `patch-storage.swift-region` | Swift region. | `string` |
| `patch-storage.postgres-connection-string` | PostgreSQL connection string. Can be left blank in charmed deployments to use Juju relations. | `string` |
| `patch-storage.s3-bucket` | S3 bucket for storing patches. | `string` |
| `patch-storage.s3-endpoint` | S3 endpoint URL. | `string` |
| `patch-storage.s3-region` | AWS region for S3. | `string` |
| `patch-storage.s3-secure` | Whether to use secure transfers. | `bool` |
| `patch-storage.s3-access-key` | AWS access key. | `string` |
| `patch-storage.s3-secret-key` | AWS secret key. | `string` |

### Patch cache

The following values configure the server's patch cache.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `patch-cache.enabled` | Whether to enable patch caching for faster delivery. | `bool` |
| `patch-cache.cache-ttl` | Time-to-live for cached patches. | `string` |
| `patch-cache.cache-size` | Maximum size of the patch cache. | `int` |

### Patch sync

The following values configure how the server synchronises patches from an upstream server.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `patch-sync.enabled` | Whether patch synchronisation is enabled. | `bool` |
| `patch-sync.id` | Unit ID. Not available in charmed deployments. | `string` |
| `patch-sync.minimum-kernel-version` | Minimum kernel version to download patches for, in `0.0.0` format. Example: `5.4.0` syncs `5.4.0` and above. | `string` |
| `patch-sync.architectures` | Comma-separated list of kernel architectures to download patches for. | `string` |
| `patch-sync.flavors` | Comma-separated list of kernel flavours to download patches for. | `string` |
| `patch-sync.interval` | Automatic synchronisation interval (for example `12h`). | `string` |
| `patch-sync.machine-count-strategy` | Strategy for reporting machine counts: `unit` reports exact counts; `bucket` reports bucketed counts using the following lower bounds: `[1-49]`, `[50-99]`, `[100-499]`, `[500-999]`, `[1000-1999]`, `[2000-4999]`, `[5000-9999]`, `[10000, ∞]`. | `oneof: unit, bucket` |
| `patch-sync.send-machine-reports` | Whether to send machine reports during synchronisation. | `bool` |
| `patch-sync.token` | Token for authenticating with an upstream Livepatch Server. | `string` |
| `patch-sync.upstream-url` | URL of the upstream server to pull patches from. | `string` |
| `patch-sync.sync-tiers` | Whether to synchronise tiers from the upstream server. | `bool` |
| `patch-sync.proxy.enabled` | Whether to use a proxy when synchronising patches. | `bool` |
| `patch-sync.proxy.http` | HTTP proxy URL. | `string` |
| `patch-sync.proxy.https` | HTTPS proxy URL. | `string` |
| `patch-sync.proxy.no-proxy` | Comma-separated list of addresses to bypass the proxy. | `string` |

### Blocklist cache

The following values configure the server's patch blocklist cache.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `patch-blocklist.enabled` | Whether to enable the blocklist cache. | `bool` |
| `patch-blocklist.refresh-interval` | Interval for refreshing the blocklist cache. | `string` |

### KPI reports

The following values configure how the server sends KPI reports. KPI reports require InfluxDB to be configured. KPIs include aggregated information on client machines such as client version and patch status.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `kpi-reports.enabled` | Whether to enable KPI reporting. | `bool` |
| `kpi-reports.interval` | Interval for submitting KPI reports. | `string` |

### Machine reports

The following values configure the server's machine report behaviour. Machine reports are stored in PostgreSQL and capture information when clients check in.

| Name | Description | Values |
| ---- | ----------- | ------ |
| `machine-reports.database.enabled` | Whether to enable machine reporting to PostgreSQL. Reports are stored in the server's PostgreSQL database. | `bool` |
| `machine-reports.database.retention-days` | Number of days to retain machine reports. | `int` |
| `machine-reports.database.cleanup-row-limit` | Maximum number of rows to delete per cleanup operation. | `int` |
| `machine-reports.database.cleanup-interval` | Interval for running report cleanup operations. | `string` |

### Cloud delay (deprecated)

The following values configure the server's cloud delay behaviour.

```{note}
The cloud delay feature is deprecated. Use the `patch-delay` and `cutoff-date` configuration options on the Livepatch Client instead.
```

| Name | Description | Values |
| ---- | ----------- | ------ |
| `cloud-delay.enabled` | Whether to delay patch releases to clients based on their cloud, region, or availability zone. | `bool` |
| `cloud-delay.default-delay-hours` | Default delay in hours for clouds, regions, or availability zones without a specified delay. | `int` |