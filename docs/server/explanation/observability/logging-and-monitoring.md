---
myst:
  html_meta:
    description: "Understand logging and monitoring for the Livepatch on-premises server including debug endpoints, Prometheus metrics, Juju log management, and security event logging."
---

(server-explanation-logging-and-monitoring)=

# Logging and monitoring

> See also: {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`, {ref}`Security overview <server-explanation-security-overview>`

The Livepatch on-premises server exposes several endpoints and mechanisms that operators can use to monitor the health and security of the deployment.

## Health and version endpoints

The Livepatch Server exposes two debug endpoints that provide information about the server's status:

- `/debug/info`: Returns the server's version information. Any monitoring solution can periodically check this endpoint as a liveness check to ensure the service is running.
- `/debug/status`: Returns information about the server's database and related services.

## Prometheus metrics

The on-premises server exposes Prometheus text-formatted metrics at the `/metrics` endpoint. These metrics can be scraped by a Prometheus instance to monitor system performance, authentication activity, request latencies, and more.

The Kubernetes charm integrates with the Canonical Observability Stack (COS) through Juju offers, using its `metrics-endpoint`, `logging`, and `grafana-dashboard` integrations. For the machine charm, Prometheus scrapes the `/metrics` endpoint directly and logs are read on each unit with `snap logs`. Metrics support aggregate monitoring but do not replace security-event records; for example, an authorizer-call count is not a count of successful logins or unique clients.

## Juju logging

Use `juju debug-log` for charm and Juju diagnostics; server application logs are in the machine's system journal or the Kubernetes application container. The log level is set by `server.log-level` for either charm and `lp.server.log-level` for the standalone snap. Both charms replace recognized credentials, authorization headers, and secret values with `***REDACTED***` in their operator logs, but this does not redact the server application's logs.

For more information on using Juju for logging, see the [Juju documentation on debug-log](https://documentation.ubuntu.com/juju/latest/reference/juju-cli/list-of-juju-cli-commands/debug-log/).

## Security logging

The Livepatch Server emits structured JSON security events, identified by `type` equal to `security`, for authentication, authorization, administrative activity, token lifecycle, and system events; the {ref}`logging schema <server-reference-logging-schema>` describes their fields. The default `info` level retains successful-authentication and authorized-administration records that `warn` drops. Log levels, forwarding, and retention are separate controls: stopping forwarding does not stop local logging or delete collected data, and machine-report database retention does not govern application logs.

For detailed guidance on configuring security event logging, Prometheus metrics, Grafana dashboards, log redaction, and centralized log forwarding, see {ref}`How to configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.