---
myst:
  html_meta:
    description: "Configure Livepatch Server log levels, local and centralized collection, Prometheus monitoring, and log retention and access controls."
---


(server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server)=

# Configure logging and monitoring for the Livepatch Server

Use this guide to collect Livepatch Server security events and monitor authentication activity, service errors, and overload. For field definitions and example records, see the {ref}`logging schema <server-reference-logging-schema>`. For monitoring concepts and health endpoints, see the [logging and monitoring explanation](/server/explanation/observability/logging-and-monitoring.md).

The Juju examples assume that the Livepatch application is named `livepatch`. Replace application names, namespaces, and pod names with your deployment's values.

(server-how-to-guides-configure-the-log-level)=

## Configure the log level

The server's log level controls the verbosity of output. Available levels are: `debug`, `info`, `warn`, `error`, `fatal`.

Use the default `info` level for normal production security visibility. Successful authentication, authorized administrative activity, and other security events are logged at `info`; selecting `warn` drops those records.

### Standalone snap

Set the log level through snap configuration:

```shell
sudo snap set canonical-livepatch-server lp.server.log-level=info
```

### Machine and Kubernetes charms

Set the log level through Juju configuration:

```shell
juju config livepatch server.log-level=info
```

Use `debug` only for bounded troubleshooting and return to `info` afterward. Review diagnostic output before sharing it because it can contain deployment or client information.

## Read local logs

### Standalone snap

Read application logs from the system journal:

```shell
sudo journalctl --no-pager -u snap.canonical-livepatch-server.livepatch.service --since "1 hour ago"
```

### Machine charm

Read the managed snap's application logs on the unit:

```shell
juju exec --unit livepatch/0 -- 'sudo journalctl --no-pager -u snap.canonical-livepatch-server.livepatch.service --since "1 hour ago"'
```

### Kubernetes charm

Read the application container's logs:

```shell
kubectl logs -n '<model-namespace>' '<livepatch-pod>' -c app --since=1h
```

For either charm, use `juju debug-log` for charm and Juju diagnostics. Do not assume that it is a complete copy of the server application's log stream.

### Find security events

Parse the application's JSON records and filter for `type` equal to `security`. Use `appid` and `event` to distinguish authentication, authorization, administrative, registration-token, and system events. Available records depend on the enabled features and severity threshold.

Collectors can add journal, Kubernetes, or Juju metadata around the application payload. Startup scripts and charm diagnostics can also emit other formats. Configure parsing for the actual payload rather than treating every collected line as a security JSON record.

See the {ref}`logging schema <server-reference-logging-schema>` for field meanings and representative identifiers.

## Configure Prometheus monitoring

Scrape the server's `/metrics` endpoint from authorized monitoring infrastructure. Keep it on a protected network or behind an appropriately restricted frontend; do not expose it publicly for convenience.

Useful exported metric families include:

| Metric family | Use |
| ------------- | --- |
| `livepatch_server_http_auth_duration_seconds` | Authentication duration, grouped by `token_type`. |
| `livepatch_server_http_authorizer_count` | Calls to token authorizers, grouped by token category in the `token` label. This is not a count of unique clients or successful authentications. |
| `livepatch_server_httpgovernor_request_overload` | Requests rejected because of overload. |
| `livepatch_server_http_request_duration_seconds` | Request duration, grouped by `method`, `handler`, and HTTP status `code`. |
| `livepatch_server_database_error_count` | Database errors. |

Histogram families expose `_bucket`, `_sum`, and `_count` series. Use security logs to investigate authentication failures rather than interpreting authorizer-call counts as authentication outcomes.

### Kubernetes charm

For existing Prometheus and Grafana applications, connect their integrations:

```shell
juju integrate livepatch:metrics-endpoint prometheus-k8s:metrics-endpoint
juju integrate livepatch:grafana-dashboard grafana-k8s:grafana-dashboard
```

### Machine charm

For an existing Grafana Agent application, connect the `cos-agent` integration:

```shell
juju integrate livepatch:cos-agent grafana-agent:cos-agent
```

This supplies monitoring configuration, dashboards, and alert rules. Configure and verify collection of the application's journal logs separately; do not assume that the relation alone forwards them.

### Standalone snap

Configure your Prometheus collector to scrape the private server listener's `/metrics` endpoint, or an approved protected proxy route. Keep scrape access restricted and apply transport protection appropriate to the network.

For all deployments, configure alerts for sustained overload, request errors, database failures, and missing monitoring data. Use thresholds appropriate to your service and investigate authentication failures through the security logs.

## Configure centralized log collection

Aggregate logs in an access-controlled, operator-managed logging service. This supports investigation across units and helps detect unexpected administration or authentication activity.

For the Kubernetes charm and an existing Loki application:

```shell
juju integrate livepatch:logging loki-k8s:logging
```

For the machine charm or standalone snap, configure your approved journal collector to forward the Livepatch service's records. Follow the collector's supported configuration procedure and avoid changing unrelated collection jobs on shared hosts.

Restrict collector credentials, use protected transport across untrusted networks, and verify that recent security records from each unit arrive at the intended destination. Air-gapped deployments should use collectors reachable within the isolated environment.

## Reduce logging or opt out of external forwarding

Severity filtering and forwarding are separate controls. Log levels are not a complete logging-off switch, and stopping forwarding does not stop local generation.

To reduce application output, for example to `error`, use the appropriate deployment method:

```shell
juju config livepatch server.log-level=error
```

```shell
sudo snap set canonical-livepatch-server lp.server.log-level=error
```

Raising the threshold to `error` or `fatal` drops both `info` and `warn` records, including successful authentication, authorized administrative activity, and many authentication failures. This reduces incident-detection and investigation evidence and is not recommended for normal production operation. Restore `info` when the exceptional need ends.

To stop Kubernetes log forwarding through the Loki integration:

```shell
juju remove-relation livepatch:logging loki-k8s:logging
```

For machine or standalone collection agents, disable the relevant Livepatch collection job using the agent's supported configuration. Do not stop a shared agent if other services depend on it. Removing the machine charm's `cos-agent` relation can affect metrics, dashboards, and alert rules as well; it is not a log-only opt-out.

Stopping forwarding removes centralized visibility and can leave evidence dependent on local retention or host availability. Disconnecting a collector is not a data-deletion procedure: manage already collected records through the destination's retention and deletion policy.

## Protect and retain log data

Logs can contain administrator identities, machine identifiers, request paths, and infrastructure details. Restrict read and write access to local logs, collectors, dashboards, exports, and backups.

Set retention and rotation policies for the system journal, Kubernetes log collection, and centralized backend as applicable. Account for exported and backed-up copies separately. Machine-report database retention settings do not control log retention.

Both the machine and Kubernetes charms replace recognized credentials, authorization headers, and secret values with `***REDACTED***` in their operator logs. This redaction does not apply to the server application's logs or cover every possible sensitive value in collector output. Review and minimize diagnostic exports before sharing them, and never rely on redaction as permission to include secrets.