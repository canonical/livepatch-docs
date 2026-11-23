---
myst:
  html_meta:
    description: "Reference for Livepatch Server structured log fields, security-event identifiers, request context, and collection boundaries."
---

(server-reference-logging-schema)=

# Logging schema

This reference describes structured records emitted by the Livepatch Server application. For collection, severity settings, forwarding, and retention procedures, see {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.

The application emits JSON records to its standard log output. Journal, Kubernetes, Juju, and collection-agent records can wrap that payload in additional metadata. Startup scripts and operator diagnostics can use other formats.

## Common fields

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `level` | String | Severity: `debug`, `info`, `warn`, `error`, or `fatal`. Records below the configured threshold are omitted. |
| `ts` | String | Timestamp encoded in ISO 8601 format. |
| `caller` | String | Diagnostic location, when included. It is not a stable event identifier. |
| `msg` | String | Human-readable log message. Do not use its exact wording as a durable parsing contract. |

Ordinary operational records do not necessarily contain security-event fields.

## Security-event fields

Security records are identified by `type` equal to `security`.

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `type` | String | Event classification; `security` identifies security-related records. |
| `appid` | String | Component or security area, such as `server.admin_authn`, `server.client_authn`, `server.client_authz`, or `server.admin`. |
| `event` | String | Event identifier. Some values include a colon followed by a reason or contextual details. |
| `description` | String | Human-readable event description. Wording and included details can vary. |

Use the component and event identifier to classify records. Do not assume that the complete `event` value is constant: it can contain identities, methods, paths, or error details.

## Optional context

Fields depend on the event and request path. Consumers should tolerate missing context and additional fields.

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `method` | String | HTTP method for a request-associated record. |
| `path` | String | Request path or event-specific path context. |
| `trace-id` | String | Request correlation identifier. It can be empty; treat it as opaque rather than assuming a UUID format. |
| `machine-id` | String | Client machine identifier associated with the operation. |
| `token-type` | String | Authentication category, not the credential value. Values vary by authentication flow. |
| `tier` | String | Tier associated with an authenticated client or synchronization request. |
| `sso-id` | String | Account or registration-group identifier when recorded. This historical field name does not require the local-token workflow to use Ubuntu SSO. |

Other operations can add diagnostic fields. Not every record contains an identity or correlation identifier, and these fields alone do not establish the integrity of a collected record.

## Representative event identifiers

This table illustrates security records; it is not an exhaustive catalog. Features that are disabled and records excluded by the log threshold do not produce the corresponding collected events.

| Identifier or pattern | Meaning |
| --------------------- | ------- |
| `authtoken_authn_success` | Local authorization-token authentication succeeded. |
| `authtoken_authn_failed:<reason>` | Local authorization-token authentication failed. |
| `machine_token_authn_success` | Machine-token authentication succeeded. |
| `machine_token_authn_failed:<reason>` | Machine-token authentication failed. |
| `macaroon_authn_success` | Administrator session authentication succeeded. |
| `macaroon_authn_failed:<reason>` | Administrator session authentication failed. |
| `authz_failed:<reason>` | Client authorization failed, for example because tier or compatibility information was invalid. |
| `authz_admin:<identity>,<method> <path>` | An authorized administrative request was recorded. |
| `authn_token_created:admin,authtoken` | An administrator's authorization-token request was processed. |
| `sys_startup`, `sys_shutdown` | Server startup or shutdown. |
| `sys_crash:<details>` | A server crash was recorded. |

Successful authentication and authorized administrative activity are normally recorded at `info`. Authentication and authorization failures commonly use `warn`. A threshold of `warn` therefore does not retain the same evidence as `info`.

## Example record

Placeholder values below represent deployment-specific diagnostic and request context:

```json
{
  "level": "info",
  "ts": "2026-01-01T12:00:00.000Z",
  "caller": "<diagnostic location>",
  "msg": "Macaroon authentication successful",
  "type": "security",
  "appid": "server.admin_authn",
  "event": "macaroon_authn_success",
  "description": "Macaroon authentication successful",
  "method": "GET",
  "path": "<request path>",
  "trace-id": "<correlation identifier>"
}
```

## Collection and data handling

Use the application's `ts` as its event timestamp and keep collector metadata distinct from application fields. Maintain appropriate clock synchronization across the deployment when correlating records.

Identities, paths, and error details can contain sensitive deployment or client information. Protect the entire record, including exports and backups. Collection, redaction, retention, and access controls are covered in the {ref}`logging how-to <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.
