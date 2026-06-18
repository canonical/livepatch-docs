---
myst:
  html_meta:
    description: "How to migrate from reactive charm to operator charm with Livepatch on-prem."
---

(server-how-to-guides-migrating-from-the-old-livepatch-charm)=

# Migrating from the old Livepatch charm

The Juju framework offered a, now deprecated, way to write charms called [reactive charms](https://documentation.ubuntu.com/juju/3.6/reference/charm/#reactive-charm). The modern framework is known as the [operator framework](https://documentation.ubuntu.com/juju/latest/reference/charm/#ops-charm).
Below, the process to identify which type of charm is running is explained.

Run `juju status` and observe the charm name and channel. The output will resemble the following.
![livepatch-status|800x31](/_static/images/2uNc2yggCQnxXj7gfkmcBVE0j2H.png)

**Machine Charm:**

- Charm name = [canonical-livepatch-server](https://charmhub.io/canonical-livepatch-server)
  - Channel = `latest/*` - Reactive charm (deprecated)
  - Channel = `ops1.x/*` - Operator charm (recommended for new deployments)

**Kubernetes Charm:**

- Charm name = [canonical-livepatch-server-k8s](https://charmhub.io/canonical-livepatch-server-k8s)
  - Channel = `latest/*` - Operator charm (recommended for new deployments)
  - Only an operator charm is available for Livepatch on K8s.

## Migrate deployment

Currently there is no way to migrate existing data to a new deployment. Existing deployments will continue to function but will no longer receive new features. It is recommended that new deployments use the operator charms.

To migrate an existing reactive charm deployment, it is suggested that you setup a new deployment and follow the steps below to migrate your configuration.

## Migrate Configuration

The new Livepatch charms have different configuration keys. Additionally, some options were removed in favour of a simpler structure. To migrate your old config to the new one, you can use this [script](https://github.com/canonical/livepatch-machine-charm/blob/main/scripts/migrate_config.py).

To run the script:

```
python3 ./converter.py -i input.yaml -o converted.yaml
```

Where the `input.yaml` is the old configuration that you want to convert. To extract it from your deployment, you can use this command:

```
juju config livepatch-server > input.yaml
```

The script would create/overwrite the output file specified by the `-o` parameter.

| | | |
| :---------------------------------: | :----------------------------------------: | :------------------------------------------------------------------------------------------------------------: |
| **Old config** | **New config** | **Notes** |
| log_level | server.log-level | Possible values:'debug', 'info', 'warn', 'error'.Default is ‘info’ |
| url_template | server.url-template | Must be in the form:“http(s)://\<hostname>:\<port>/v1/patches/{filename}" |
| psql_dbname | **NA** | |
| psql_roles | **NA** | |
| blocklist_cache_refresh | patch-blocklist.refresh-interval | Make sure that patch-blocklist.enabled is set to true |
| contract_server_url | contracts.url | |
| contract_server_user | contracts.user | |
| contract_server_password | contracts.password | |
| concurrency_limit | server.concurrency-limit | |
| burst_limit | server.burst-limit | |
| is_cloud_delay_enabled | cloud-delay.enabled | |
| cloud_delay_default_delay_hours | cloud-delay.default-delay-hours | |
| dbconn_max | database.connection-pool-max | |
| dbconn_max_lifetime | database.connection-lifetime-max | |
| patch_sync_enabled | patch-sync.enabled | |
| patch_cache_ttl | patch-cache.cache-ttl | |
| patch_cache_size | patch-cache.cache-size | |
| patch_cache_on | patch-cache.enabled | |
| http_proxy | patch-sync.proxy.http | Make sure that **patch-sync.proxy.enabled** is set to true. |
| https_proxy | patch-sync.proxy.https | |
| no_proxy | patch-sync.proxy.no-proxy | |
| | | |
| event_bus_client_cert | machine-reports.event-bus.client-cert | Make sure: machine-reports.event-bus.enabled is set to true |
| event_bus_client_key | machine-reports.event-bus.client-key | |
| event_bus_ca_cert | machine-reports.event-bus.ca-cert | |
| event_bus_brokers | machine-reports.event-bus.brokers | |
| auth_lp_teams | auth.sso.teams | Make sure this is set to true: auth.sso.enabled |
| auth_sso_public_key | auth.sso.public-key | |
| auth_sso_location | auth.sso.url | |
| auth_basic_users | auth.basic.users | Make sure that auth.basic.enable is set to true |
| | | |
| swift_container_name | patch-storage.swift-container | |
| swift_auth_url | patch-storage.swift-auth-url | |
| swift_username | patch-storage.swift-username | |
| swift_apikey | patch-storage.swift-api-key | |
| swift_tenant_name | patch-storage.swift-tenant | |
| swift_region_name | patch-storage.swift-region | |
| swift_domain_name | patch-storage.swift-domain | |
| patchstore | patch-storage.type | default: filesystem The available options are: - filesystem - swift - postgres - s3 |
| storage_path | patch-storage.filesystem-path | Path defaults to /var/snap/canonical-livepatch-server/common/patches |
| sync_token | patch-sync.token | |
| sync_upstream | patch-sync.upstream-url | |
| sync_identity | **NA** | |
| sync_upstream_tier | **NA** | Upstream tier to download patch snapshots from. Default value is "updates". |
| sync_interval | patch-sync.interval | |
| sync_tier | **NA** | Tier to assign downloaded patches to. Defaults to "edge". |
| sync_flavors | patch-sync.flavors | |
| sync_architectures | patch-sync.architectures | |
| sync_minimum_kernel_version | patch-sync.minimum-kernel-version | |
| report_retention | machine-reports.database.retention-days | |
| report_cleanup_interval | machine-reports.database.cleanup-interval | |
| report_cleanup_row_limit | machine-reports.database.cleanup-row-limit | |
| influxdb_url | influx.url | |
| influxdb_token | influx.token | |
| influxdb_bucket | influx.bucket | |
| influxdb_organization | influx.organization | |
| s3_bucket | patch-storage.s3-bucket | |
| s3_endpoint | patch-storage.s3-endpoint | |
| s3_region | patch-storage.s3-region | |
| s3_access_key_id | patch-storage.s3-access-key | |
| s3_secret_key | patch-storage.s3-secret-key | |
| s3_secure | patch-storage.s3-secure | |
| profiler_enabled | profiler.enabled | |
| profiler_server_address | profiler.server_address | |
| profiler_hostname | profiler.hostname | |
| profiler_sample_rate | profiler.sample_rate | |
| profiler_upload_rate | profiler.upload_rate | |
| profiler_mutex_profile_fraction | profiler.mutex_profile_fraction | |
| profiler_block_profile_rate | profiler.block_profile_rate | |
| profiler_profile_allocations | profiler.profile_allocations | |
| profiler_profile_inuse | profiler.profile_inuse | |
| profiler_profile_mutexes | profiler.profile_mutexes | |
| profiler_profile_blocks | profiler.profile_blocks | |
| profiler_profile_goroutines | profiler.profile_goroutine | |
