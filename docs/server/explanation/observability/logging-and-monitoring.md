---
myst:
  html_meta:
    description: "Logging and monitoring - learn about this topic in Livepatch on-prem."
---


(server-explanation-logging-and-monitoring)=

# Logging and Monitoring

Monitoring of the Livepatch server can be most easily done by setting up monitoring on one or more endpoints. Livepatch server exposes two endpoints, in particular, `/debug/info` and `/debug/status`, that provide information on the server’s version and the server’s database/related services, respectively. Any monitoring solution can periodically check `/debug/info` as a liveliness check to ensure the service is running.

The on-prem server also exposes Prometheus text-based formatted metrics available from a /metrics endpoint which can be used to monitor the system.

When deploying with Juju, debug logs from all deployed applications are available through the `juju debug-logs` command. The server's log level can be configured via the `log_level` configuration option. A full list of log levels are available on the Configure logging and monitoring [page](/server/how-to-guides/operations/configure-logging-and-monitoring.md#configure-the-log-level).

Further information on the use of Juju for logging can be obtained from the Juju [documentation](https://documentation.ubuntu.com/juju/latest/reference/juju-cli/list-of-juju-cli-commands/debug-log/).

## Security logging

For detailed guidance on configuring security event logging, Prometheus metrics, Grafana dashboards, log redaction, and centralized log forwarding, see [How to configure logging and monitoring](/server/how-to-guides/operations/configure-logging-and-monitoring.md).
