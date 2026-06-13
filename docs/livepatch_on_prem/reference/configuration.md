---
myst:
  html_meta:
    description: "Configuration - technical reference for Livepatch on-prem server."
---


(livepatch_on_prem-reference-configuration)=

# Configuration

This document provides the configuration options for the Livepatch server.

```{note}
The configuration below applies to the Livepatch Server operator charms and Server snap. 
For our reactive charm (or if your deployment config doesn't match the below), please see [here](/livepatch_on_prem/how-to-guides/migrate-from-reactive-charm-to-operator-charm.md).
```

## Setting config

Depending on your deployment you may be using Juju + charms or a standalone Snap to deploy the Livepatch server. How you setup config will differ slightly between the two.

Default and example values are available on the respective [machine](https://charmhub.io/canonical-livepatch-server/configurations?channel=ops1.x/stable) and [K8s](https://charmhub.io/canonical-livepatch-server-k8s/configurations) charm config pages.

### Juju

The config values in the table below map directly to the config options exposed by the Livepatch charms (except where otherwise stated).

Assuming the Livepatch server has been deployed with the alias `livepatch`, to change a config value run:

```
juju config livepatch <key>=<value>
# E.g. to enable basic auth
juju config livepatch auth.basic.enabled=true
```

See the [Juju docs](https://documentation.ubuntu.com/juju/latest/reference/juju-cli/list-of-juju-cli-commands/config/) for all the ways you can apply config.

### Snap

The config values in the table below map directly to the config values accepted by the Livepatch server snap. An additional value must be added to all commands as shown below.

To change a config value run:

```
sudo snap set canonical-livepatch-server lp.<key>=<value>
# E.g. to enable basic auth
sudo snap set canonical-livepatch-server lp.auth.basic.enabled=true
```

## Config

The following sections describes what configuration values are available.

### Server config

| Name | Description | Value(s) |
| - | - | - |
| `server.log-level` | Log level for the server | `debug, info, warn, error, dpanic, panic, fatal` |
| `server.redirect_downloads` | Whether to redirect patch download requests to the URL configured in the `server.url-template` option. | `bool` |
| `server.url-template` | The template URL to redirect clients for patch downloads. For example: `https://my-file-server.com/{filename}`. | `string` |
| `server.server-address` | Listen address for the server. | `url` |
| `server.concurrency-limit` | Maximum number of API requests to serve concurrently. | `integer` |
| `server.burst-limit` | The queue limit, roughly equals `concurrency-burst-limit`. | `integer` |
| `server.is-leader` | In multi-server deployments, determine if this is a leader unit. Not available for charmed deployments. | `bool`. |
| `server.is-hosted` | Enable configuration blocks specific to Canonical's hosted configuration for livepatch. | `bool` |

### Admin Authentication

The following values configure authentication to the server's admin endpoints.
Besides basic auth, only Ubuntu SSO auth is supported.

Some notes on this section:

- SSO Teams represent Launchpad teams.
- Basic auth can be a comma separated list, see [here](/livepatch_on_prem/how-to-guides/setup-administration-tool.md#password-authentication) for more info.
- Basic auth passwords *must* be bcrypt hashed.

| Name | Description | Value(s) |
| - | - | - |
| `auth.basic.enabled` | Whether or not to enable basic auth. | `bool`|
| `auth.basic.users` | A comma separated list of user objects. | `<user1>:<bcrypt hashed password>, <user2>:<bcrypt hashed password>` |
| `auth.sso.enabled` | Whether or not to enable Ubuntu SSO auth. | `bool`|
| `auth.sso.teams` | SSO Auth configuration. | `https://launchpad.net/~team-1,https://launchpad.net/~team-2` |
| `auth.sso.url` | URL to access for SSO auth. | `login.ubuntu.com` |
| `auth.sso.public-key` | Public key for the auth server. Can be a file path or the key. | `string` |

### Ubuntu Pro

The following values configure how the server interacts with the Ubuntu Pro backend (also called the contracts server) for authenticating clients.
This is useful for Canonical's hosted Livepatch server and airgapped deployments.

| Name | Description | Value(s) |
| - | - | - |
| `contracts.enabled` | Whether to connect to the contracts service. | `bool` |
| `contracts.url` | URL of the contracts server. | `string` |
| `contracts.user` | Basic auth user. | `string` |
| `contracts.password` | Basic auth pass. | `string` |

### Database

The following values configure how the server interacts with its database.

| Name | Description | Value(s) |
| - | - | - |
| `database.connection-string` | PostgreSQL connection string (unavailable for charmed deployments, handled with Juju relations). | `string` |
| `database.connection-pool-max` | Max pool for connections. | `int` |
| `database.connection-lifetime-max` | Max lifetime of connections. | `int` |
| `database.work-mem` | The maximum amount of memory in MB available for each query operation. | `int` |

### Influx

The following values configure how the server interacts with InfluxDB, used for sending aggregated KPIs.

| Name | Description | Value(s) |
| - | - | - |
| `influx.enabled` | Whether to enable influx KPI reporting (hosted). | `bool` |
| `influx.url` | URL of the Influx server. | `string` |
| `influx.token` | Auth token. | `string` |
| `influx.bucket` | Bucket to use. | `string` |
| `influx.ping_bucket` | Bucket to use for sending patch ping data, for example, under a different retention policy. If empty, this is set to the value of `influx.bucket`. | `string` |
| `influx.organization` | Org where bucket resides. | `string` |

### TimescaleDB

The following values configure how the server interacts with its TimescaleDB instance, used for storing time-series patch ping metrics.
TimescaleDB runs as a PostgreSQL extension on a dedicated PostgreSQL instance separate from the main database.
TimescaleDB can be enabled along side or can replace InfluxDB for KPI metrics.

| Name | Description | Value(s) |
| - | - | - |
| `timescale-db.enabled` | Whether to enable the TimescaleDB store for KPI metrics. | `bool` |
| `timescale-db.connection-string` | Connection string for the TimescaleDB PostgreSQL instance. | `string` |
| `timescale-db.connection-pool-max` | Maximum number of connections in the pool (1–1000). | `int` |
| `timescale-db.connection-lifetime-max` | Maximum lifetime of a connection before it is recycled. | `duration` |
| `timescale-db.work-mem` | The maximum amount of memory in MB available for each query operation. | `int` |
| `timescale-db.flush-timeout` | How long the store waits before flushing buffered writes. Defaults to 1 minute if unset. | `duration` |

### Patch Storage

The following values configure how the server interacts with its patch storage.\
See our [how-to](/livepatch_on_prem/explanation/patch-storage/index.md) on patch storage.

| Name | Description | Value(s) |
| - | - | - |
| `patch-storage.type` | File storage type to use for on-prem deployment patch syncs | `oneof: filesystem,swift,postgres,s3` |
| `patch-storage.filesystem-path` | File path to directory to use for storage | `string` |
| `patch-storage.swift-username` | User of account. | `string` |
| `patch-storage.swift-api-key` | Auth API key. | `string` |
| `patch-storage.swift-auth-url` | Auth Url. | `string` |
| `patch-storage.swift-domain` | Swift domain to connect to. | `string` |
| `patch-storage.swift-tenant` | Swift tenancy. | `string` |
| `patch-storage.swift-container` | Swift container bucket. | `string` |
| `patch-storage.swift-region` | Swift region. | `string` |
| `patch-storage.postgres-connection-string` | PostgreSQL connection string (can be left blank in charmed deployments to use Juju relations). | `string` |
| `patch-storage.s3-bucket` | S3 Bucket to store patches. | `string` |
| `patch-storage.s3-endpoint` | S3 endpoint. | `string` |
| `patch-storage.s3-region` | AWS Region for S3. | `string` |
| `patch-storage.s3-secure` | Whether to perform secure transfers. | `bool` |
| `patch-storage.s3-access-key` | AWS Access key. | `string` |
| `patch-storage.s3-secret-key` | AWS Secret key. | `string` |

### Patch Cache

The following values configure the server's patch cache.

| Name | Description | Value(s) |
| - | - | - |
| `patch-cache.enabled` | Whether or not to cache patches for quicker delivery. | `bool` |
| `patch-cache.cache-ttl` | TTL of patches in cache. | `string` |
| `patch-cache.cache-size` | Maximum size of caching for patches. | `int` |

### Patch Sync

The following values configure how the server syncs patches from an upstream server.

| Name | Description | Value(s) |
| - | - | - |
| `patch-sync.enabled` | Whether patch syncs are enabled. | `bool` |
| `patch-sync.id` | ID of unit (not available in charmed deployments). | `string` |
| `patch-sync.minimum-kernel-version` | A minimum kernel version of format "0.0.0" denoting the lowest kernel version to download patches for. For example, "5.4.0" will sync "5.4.0" and up. | `string` |
| `patch-sync.architectures` | Comma-separated list of kernel architectures to download patches for. | `string` |
| `patch-sync.flavors` | Comma-separated list of kernel flavors to download patches for. | `string` |
| `patch-sync.interval` | Automatic sync interval e.g. 12h. | `string` |
| `patch-sync.machine-count-strategy` | Define the way sync reports the machine counts, either by units or by buckets. On on-prem instances the counts are bucketed and the value reported is given by lower bound of the following buckets: `[1-49]`, `[50-99]`, `[100-499]`, `[500-999]`, `[1000-1999]`, `[2000-4999]`, `[5000-9999]`, `[10000, ∞]`. | `oneof: unit,bucket` |
| `patch-sync.send-machine-reports` | Whether or not to send machine reports. | `bool` |
| `patch-sync.token` | Token used to authorise with an upstream Livepatch server. | `string` |
| `patch-sync.upstream-url` | The upstream server to pull patches from. | `string` |
| `patch-sync.sync-tiers` | Enable syncing tiers from upstream server. | `bool` |
| `patch-sync.proxy.enabled` | Enable use of a proxy when syncing patches. | `bool` |
| `patch-sync.proxy.http` | HTTP Proxy. | `string` |
| `patch-sync.proxy.https` | HTTPS Proxy. | `string` |
| `patch-sync.proxy.no-proxy` | Comma separated list of addresses that should not go through the proxy. | `string` |

### Blocklist Cache

The following values configure the server's patch blocklist cache.

| Name | Description | Value(s) |
| - | - | - |
| `patch-blocklist.enabled` | Whether or not to enable the blocklist cache. | `bool` |
| `patch-blocklist.refresh-interval` | How often to refresh the blocklist cache. | `string` |

### KPI Reports

The following values configure how the server sends KPI reports. This requires Influx to be setup.
KPIs include aggregated information on client machines e.g. the client version, patch status, etc.

| Name | Description | Value(s) |
| - | - | - |
| `kpi-reports.enabled` | Whether or not to enable KPI reporting. | `bool` |
| `kpi-reports.interval` | How often to submit reports. | `string` |

### Machine reports

The following values configure the server's behavior with machine reports.
Machine reports are stored in PostgreSQL and store information when client's check-in.

<!---
The config values for `event-bus` are intended for Canonical internal use only.
I've left the event-bus config options commented out since they are not relevant to users.
-->

| Name | Description | Value(s) |
| - | - | - |
| `machine-reports.database.enabled` | Whether or not to enable machine reporting to postgres. Reports are stored in the server's postgres store. | `bool` |
| `machine-reports.database.retention-days` | Retention for the given reports. | `int` |
| `machine-reports.database.cleanup-row-limit` | Row limit for each cleanup operation. | `int` |
| `machine-reports.database.cleanup-interval` | How often to perform cleanups. | `string` |

## Cloud delay

The following values configure the server's behavior with cloud-delays.

```{note}
The cloud delay feature is deprecated in favor of the patch-delay and cutoff-date configuration in Livepatch Client.
```

| Name | Description | Value(s) |
| - | - | - |
| `cloud-delay.enabled` | Enable the server to delay the release of patches to clients based on their cloud/region/az | `bool` |
| `cloud-delay.default-delay-hours` | Default delay hours for clouds/regions/azs without predefined delay hours | `int` |
