---
myst:
  html_meta:
    description: "Configure Livepatch Server log levels, local and centralized collection, Prometheus monitoring, and log retention and access controls."
---


(server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server)=

# Configure logging and monitoring for the Livepatch Server

The Livepatch Server provides structured security event logging and Prometheus metrics for monitoring. For general monitoring concepts and debug endpoints, see the [Logging and Monitoring explanation](/server/explanation/observability/logging-and-monitoring.md). The Juju examples assume that the Livepatch application is named `livepatch`; replace application names, namespaces, and pod names with your deployment's values.

(server-how-to-guides-configure-the-log-level)=

## Configure the log level

The server's log level controls the verbosity of output. Available levels are: `debug`, `info`, `warn`, `error`, `fatal`.

Use the default `info` level in production. Successful authentication, authorized administrative activity, and other security events are logged at `info`, so `warn` drops those records. Use `debug` only for bounded troubleshooting.

### Snap

Set the log level through snap configuration:

```shell
sudo snap set canonical-livepatch-server lp.server.log-level=info
```

### Charm

Set the log level through Juju configuration:

```shell
juju config livepatch server.log-level=info
```

## Configure Prometheus monitoring

The server exposes metrics at the `/metrics` endpoint in Prometheus format. Key security-relevant metrics include:

| Metric | Description |
| :---- | :---- |
| `livepatch_server_http_auth_duration_seconds` | Authentication duration, grouped by `token_type`. |
| `livepatch_server_http_authorizer_count` | Calls to token authorizers, grouped by the `token` label. This is not a count of unique clients or successful authentications. |
| `livepatch_server_httpgovernor_request_overload` | Requests rejected because of overload. |
| `livepatch_server_http_request_duration_seconds` | Request duration, grouped by `method`, `handler`, and HTTP status `code`. |
| `livepatch_server_database_error_count` | Database errors. |

For the machine charm and standalone snap, configure Prometheus to scrape the private server listener's `/metrics` endpoint. For the Kubernetes charm, send metrics to COS as described below. In every deployment, keep `/metrics` on a protected network rather than exposing it publicly, and alert on sustained overload, request errors, database failures, and missing monitoring data.

## Send Kubernetes charm metrics and logs to COS

For the Kubernetes charm, aggregate metrics and logs in the [Canonical Observability Stack (COS)](https://canonical.com/observability/docs/track-3.1/), and query logs through Grafana rather than reading individual units. COS runs on Kubernetes in its own model and exposes Prometheus, Loki, and Grafana through Juju offers. Offer names depend on your COS deployment. From the Livepatch model, list them:

```shell
juju find-offers -m <cos-controller>:admin/<cos-model>
```

Consume the offers and integrate Livepatch with them:

```shell
juju consume <cos-controller>:admin/<cos-model>.prometheus-metrics-endpoint
juju consume <cos-controller>:admin/<cos-model>.loki-logging
juju consume <cos-controller>:admin/<cos-model>.grafana-dashboards
juju integrate livepatch:metrics-endpoint prometheus-metrics-endpoint
juju integrate livepatch:logging loki-logging
juju integrate livepatch:grafana-dashboard grafana-dashboards
```

Restrict access to COS and check in Grafana that recent security records from each unit arrive in Loki.

## Read machine charm and standalone snap logs

For the machine charm, read the server logs on each Livepatch unit:

```shell
juju ssh livepatch/0 sudo snap logs canonical-livepatch-server
```

For the standalone snap, read the logs on the host:

```shell
sudo snap logs canonical-livepatch-server
```

Add `-n=<lines>` to show more lines, or `-f` to follow new entries. To aggregate these logs, configure your journal collector to forward the Livepatch service's records.

To find security events, parse the JSON records, filter for `type` equal to `security`, and use `appid` and `event` to classify them; see the {ref}`logging schema <server-reference-logging-schema>`. Collectors can wrap records in journal, Kubernetes, or Juju metadata, and startup scripts and charm diagnostics use other formats. `juju debug-log` shows charm and Juju diagnostics; do not treat it as a complete copy of the server application's logs.

## Reduce logging or opt out of forwarding

Log levels and forwarding are separate controls: raising the log level does not turn logging off completely, and stopping forwarding does not stop local logging. To reduce output, for example to `error`, run `juju config livepatch server.log-level=error` for a charm, or `sudo snap set canonical-livepatch-server lp.server.log-level=error` for the standalone snap.

To stop forwarding Kubernetes charm logs to COS, remove the Loki integration:

```shell
juju remove-relation livepatch:logging loki-logging
```

For the machine charm or standalone snap, disable the Livepatch job in your journal collector rather than stopping a shared collector.

Raising the level to `error` or `fatal` drops `info` and `warn` records, including successful authentication, authorized administrative activity, and many authentication failures. This reduces the evidence available to detect and investigate security incidents and is not recommended for normal production operation; restore `info` when the need ends. Stopping forwarding removes centralized visibility and can leave evidence dependent on local retention or host availability. It does not delete collected records; manage those through the destination's retention policy.

## Protect and retain log data

Logs can contain administrator identities, machine identifiers, request paths, and infrastructure details. Restrict access to local logs, collectors, dashboards, exports, and backups, and set retention and rotation for the journal, Kubernetes log collection, and centralized backend. Machine-report database retention settings do not control log retention. Both charms replace recognized credentials, authorization headers, and secret values with `***REDACTED***` in their operator logs. This does not apply to the server application's logs or cover every sensitive value, so review diagnostic exports before sharing them.