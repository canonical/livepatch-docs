---
myst:
  html_meta:
    description: "Reference for Livepatch Server structured log fields, security-event identifiers, request context, and collection boundaries."
---

(server-reference-logging-schema)=

# Logging schema

The Livepatch Server application emits JSON records to its standard log output. Collectors can wrap records in journal, Kubernetes, or Juju metadata, and fields vary by event, so tolerate missing and additional fields. Identities, paths, and error details can be sensitive; protect complete records, including exports and backups, as described in {ref}`Configure logging and monitoring <server-how-to-guides-configure-logging-and-monitoring-for-livepatch-server>`.

## Example record

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

## Fields

| Common field | Type | Meaning |
| ------------ | ---- | ------- |
| `level` | String | Severity: `debug`, `info`, `warn`, `error`, or `fatal`. Records below the configured threshold are omitted. Successful authentication and authorized administrative activity are normally `info`; failures commonly use `warn`. |
| `ts` | String | Timestamp encoded in ISO 8601 format. |
| `caller` | String | Diagnostic location, when included. It is not a stable event identifier. |
| `msg` | String | Human-readable log message. Do not rely on its exact wording. |

| Security-event field | Type | Meaning |
| -------------------- | ---- | ------- |
| `type` | String | Event classification; `security` identifies security-related records. |
| `appid` | String | Component or security area, such as `server.admin_authn`, `server.client_authn`, `server.client_authz`, `server.client`, or `server.admin`. |
| `event` | String | Event identifier. Some values include a colon followed by a reason or details such as identities, methods, paths, or errors. |
| `description` | String | Human-readable event description. Wording and included details can vary. |

| Optional context field | Type | Meaning |
| ---------------------- | ---- | ------- |
| `method`, `path` | String | HTTP method and request path, or event-specific path context. |
| `trace-id` | String | Request correlation identifier. It can be empty; treat it as opaque. |
| `machine-id` | String | Client machine identifier associated with the operation. |
| `token-type` | String | Authentication category, not the credential value. |
| `tier` | String | Tier associated with an authenticated client or synchronization request. |
| `sso-id` | String | Account or registration-group identifier when recorded. The historical name does not mean that the local-token workflow uses Ubuntu SSO. |

## Event identifiers

| Representative event identifier | Meaning |
| ------------------------------- | ------- |
| `authtoken_authn_success`, `authtoken_authn_failed:<reason>` | Local authorization-token authentication succeeded or failed. |
| `machine_token_authn_success`, `machine_token_authn_failed:<reason>` | Machine-token authentication succeeded or failed. |
| `macaroon_authn_success`, `macaroon_authn_failed:<reason>` | Administrator session authentication succeeded or failed. |
| `basicauth_authn_success`, `basicauth_authn_failed:<reason>` | Administrator password authentication succeeded or failed. |
| `resource_token_authn_success`, `resource_token_authn_failed:<reason>` | Resource-token authentication succeeded or failed. |
| `authz_failed:<reason>` | Client authorization failed, for example because tier or compatibility information was invalid. |
| `authz_admin:<identity>,<method> <path>` | An authorized administrative request was recorded. |
| `authn_token_created:<details>`, `authn_token_updated:<details>`, `authn_token_deleted:<details>` | A token was created, updated, or deleted. Details name the actor and token type, such as `admin,auth_sync_token`. |
| `sys_startup`, `sys_shutdown`, `sys_crash:<details>` | Server startup, shutdown, or crash. |
