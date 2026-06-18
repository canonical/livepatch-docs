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

The Livepatch server exposes two debug endpoints that provide information about the server's status:

- `/debug/info`: Returns the server's version information. Any monitoring solution can periodically check this endpoint as a liveness check to ensure the service is running.
- `/debug/status`: Returns information about the server's database and related services.

## Prometheus metrics

The on-premises server exposes Prometheus text-formatted metrics at the `/metrics` endpoint. These metrics can be scraped by a Prometheus instance to monitor system performance, authentication activity, request latencies, and more.

When deploying with the charm, the `metrics-endpoint` integration enables Prometheus scraping, and the `grafana-dashboard` integration ships a pre-built Grafana dashboard tracking authentication metrics, token usage by type, contracts service errors, and request performance.

## Juju logging

When deploying with Juju, debug logs from all deployed applications are available through the `juju debug-logs` command. The server's log level can be configured via the `log_level` configuration option. For a full list of log levels, see {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.

For more information on using Juju for logging, see the [Juju documentation on debug-log](https://documentation.ubuntu.com/juju/latest/reference/juju-cli/list-of-juju-cli-commands/debug-log/).

## Security logging

The Livepatch server logs all authentication successes and failures, authorization decisions, token lifecycle events, admin activity, and system events as structured JSON in compliance with OWASP logging standards.

For detailed guidance on configuring security event logging, Prometheus metrics, Grafana dashboards, log redaction, and centralized log forwarding, see {ref}`How to configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.