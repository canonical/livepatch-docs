---
myst:
  html_meta:
    description: "How to configure logging and monitoring with Livepatch on-prem."
---


(livepatch_on_prem-how-to-guides-configure-logging-and-monitoring-for-livepatch-server)=

# Configure logging and monitoring for Livepatch server

The Livepatch Server provides structured security event logging and Prometheus metrics for monitoring. This document describes how to configure and consume these security-relevant features. For general monitoring concepts and debug endpoints, see the existing [Logging and Monitoring explanation](/server/explanation/logging-and-monitoring.md).

(livepatch_on_prem-how-to-guides-configure-the-log-level)=

## Configure the log level

The server's log level controls the verbosity of output. Available levels are: `debug`, `info`, `warn`, `error`, `fatal`.

For production deployments, use `info` or `warn` to capture security events without excessive noise.

### Snap

Set the log level through snap configuration:

```shell
sudo snap set canonical-livepatch-server log-level=info
```

### Charm

Set the log level through Juju configuration:

```shell
juju config canonical-livepatch-server-k8s server.log-level=info
```

## Security events

All security events are emitted to the standard logger output as structured JSON. Each event includes a `type: "security"` field for filtering. The following categories of security events are logged:

- **Authentication successes and failures** for all authentication methods (Basic Auth, Macaroons, machine tokens, resource tokens, sync tokens, webhook tokens)
- **Authorization failures** (tier mismatches, invalid affordances, unauthorized access attempts)
- **Token lifecycle events** (creation, deletion, and updates of authentication tokens)
- **Administrative activity** (all admin API requests with identity and endpoint information)
- **System events** (startup, shutdown, crash, and OS signal handling)

## Configure Prometheus monitoring

The server exposes metrics at the `/metrics` endpoint in Prometheus format. Key security-relevant metrics include:

| Metric | Description |
| :---- | :---- |
| `AuthDuration` | Time taken to authenticate requests, by token type |
| `AuthorizerCount` | Count of authentication attempts, by token type |
| `RequestOverloadCounter` | Requests dropped due to server overload |
| `RequestDuration` | HTTP request latency, by method, handler, and status code |
| `DatabaseErrorCount` | Total database errors |

Configure Prometheus to scrape the `/metrics` endpoint. For charm deployments, the charm provides a `metrics-endpoint` integration for Prometheus scraping and a `grafana-dashboard` integration that ships a pre-built Grafana dashboard:

```shell
juju integrate canonical-livepatch-server-k8s prometheus-k8s:metrics-endpoint
juju integrate canonical-livepatch-server-k8s grafana-k8s:grafana-dashboard
```

The Grafana dashboard includes panels for authentication timing by token type, 24-hour token usage counts, contracts service errors, and request throughput.

For centralized log collection, the charm supports forwarding logs to Loki:

```shell
## Juju >= 3.4 (direct log forwarding)
juju integrate canonical-livepatch-server-k8s loki-k8s:logging

## Juju < 3.4 (Promtail-based log proxy)
juju integrate canonical-livepatch-server-k8s loki-k8s:log-proxy
```

## Log structure

Log entries follow this structure:

```json
{
  "level": "warn",
  "ts": "2025-10-01T10:45:41.142Z",
  "caller": "httpauth/basic.go:26",
  "msg": "...",
  "type": "security",
  "appid": "server.admin_authn",
  "event": "basicauth_authn_failed:user_not_found",
  "description": "User not found",
  "method": "POST",
  "path": "/v1/admin/auth-token",
  "trace-id": "39e28f24-..."
}
```

### Snap

Application logs are captured by systemd-journald. To view security logs:

```shell
sudo journalctl -u snap.canonical-livepatch-server.livepatch.service | grep '"type":"security"'
```

### Charm (Kubernetes)

Container logs are available through standard Kubernetes log collection:

```shell
kubectl logs -n <model-name> <livepatch-pod> | grep '"type":"security"'
```

The server also writes logs to `/var/log/livepatch` inside the container, with logrotate configured for daily rotation (3 files retained, 10MB size trigger, compression enabled).

When integrated with Loki via the `logging` or `log-proxy` relations, logs are forwarded automatically and can be queried through Grafana.

## Charm log redaction

The charm itself (not the server binary) includes a log redaction module that scrubs sensitive data from charm operator logs. The following patterns are redacted before logs reach any handler:

- Database connection URIs with embedded credentials (e.g., `postgresql://user:password@host/db`)
- HTTP Authorization headers (`Bearer` and `Basic` tokens)
- Key-value pairs where the key implies a secret (e.g., `password=`, `token=`, `api-key=`, `credentials=`)
- Specific environment variables: `LP_CONTRACTS_PASSWORD`, `LP_PATCH_SYNC_TOKEN`, `LP_PATCH_STORAGE_S3_SECRET_KEY`, `LP_PATCH_STORAGE_S3_ACCESS_KEY`, `LP_PATCH_STORAGE_SWIFT_API_KEY`, `LP_AUTH_BASIC_USERS`, `LP_AUTH_SSO_PUBLIC_KEY`, `LP_DATABASE_CONNECTION_STRING`, and others

All redacted values are replaced with `***REDACTED***` in log output.

## Risks of disabling logging

Reducing the log level to `error` or `fatal` will suppress security event logs at the `info` and `warn` levels, which includes authentication successes, token lifecycle events, and administrative activity. This significantly reduces visibility into security-relevant operations and is not recommended for production deployments.
