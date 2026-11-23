---
myst:
  html_meta:
    description: "Understand logging and monitoring for the Livepatch on-premises server including debug endpoints, Prometheus metrics, Juju log management, and security event logging."
---

(server-explanation-logging-and-monitoring)=

# Logging and monitoring

> See also: {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`, {ref}`Logging schema <server-reference-logging-schema>`, {ref}`Security overview <server-explanation-security-overview>`

The Livepatch on-premises server exposes several endpoints and mechanisms that operators can use to monitor the health and security of the deployment.

## Health and version endpoints

The Livepatch Server exposes two debug endpoints that provide information about the server's status:

- `/debug/info`: Returns the server's version information. Any monitoring solution can periodically check this endpoint as a liveness check to ensure the service is running.
- `/debug/status`: Returns information about the server's database and related services.

## Prometheus metrics

The on-premises server exposes Prometheus text-formatted metrics at the `/metrics` endpoint. These metrics can be scraped by a Prometheus instance to monitor system performance, authentication activity, request latencies, and more.

The Kubernetes charm provides `metrics-endpoint` and `grafana-dashboard` integrations. The machine charm provides monitoring configuration, dashboards, and alert rules through `cos-agent`. These are distinct integration paths; the Kubernetes relation names do not apply to the machine charm.

Metrics support aggregate monitoring but do not replace security-event records. For example, an authorizer-call count is not a count of successful logins or unique clients.

## Juju logging

Use `juju debug-log` for charm and Juju diagnostics. Application logs are also available through the machine's system journal or the Kubernetes application container, depending on deployment type.

The application setting is `server.log-level` for either charm and `lp.server.log-level` for the standalone snap. Severity filtering determines which application records are emitted; it is separate from collecting or forwarding those records.

Both the machine and Kubernetes charms replace recognized credentials, authorization headers, and secret values with `***REDACTED***` in their operator logs. This does not redact the server application's log stream or guarantee that all sensitive values are removed. Review diagnostic exports before sharing them.

For more information on using Juju for logging, see the [Juju documentation on debug-log](https://documentation.ubuntu.com/juju/latest/reference/juju-cli/list-of-juju-cli-commands/debug-log/).

## Security logging

The Livepatch Server emits structured security events for authentication, authorization, administrative activity, registration-token issuance, and system lifecycle events. Coverage depends on enabled features and the configured severity threshold. The `type` field identifies security records; the {ref}`logging schema <server-reference-logging-schema>` describes their fields and representative identifiers.

The default `info` threshold retains informational security events. Raising the threshold to `warn` loses successful-authentication and authorized-administration records; higher thresholds lose additional failure evidence.

## Collection and retention

Centralized collection supports investigation across units, but local generation, forwarding, and retention are separate controls. Stopping forwarding does not stop local logging or constitute deletion of already collected data.

Operators manage journal, Kubernetes, collector, export, and backup retention according to their deployment policies. Machine-report database retention does not govern application logs.

For detailed guidance on configuring security event logging, Prometheus metrics, Grafana dashboards, log redaction, and centralized log forwarding, see {ref}`How to configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.