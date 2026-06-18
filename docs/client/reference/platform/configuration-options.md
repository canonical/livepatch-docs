---
myst:
  html_meta:
    description: "Configuration reference for the Livepatch Client, including proxy settings, server URL, check intervals, and TLS enforcement options."
---

(client-reference-livepatch-client-configuration-options)=

# Livepatch Client configuration options

The following configuration options are available for the Livepatch Client. See the [how-to guide on configuring the Livepatch Client](/client/how-to-guides/configuration/configure-livepatch-client.md) for instructions on applying these settings.

| Key | Data type | Description | Default value |
| --- | --------- | ----------- | ------------- |
| `http-proxy` | string | Value passed as `HTTP_PROXY`. Overrides `/etc/environment`. | Empty |
| `https-proxy` | string | Value passed as `HTTPS_PROXY`. Overrides `/etc/environment`. | Empty |
| `no-proxy` | string | Value passed as `NO_PROXY`. Overrides `/etc/environment`. | Empty |
| `remote-server` | string | URL of the Livepatch Server. | `https://livepatch.canonical.com` |
| `ca-certs` | string | Custom CA root certificates encoded as PEM. | Empty |
| `dial-timeout` | string | Timeout for opening TCP connections. Allowed units: `s`, `m`, `h`. | `12s` |
| `check-interval` | integer | Interval in minutes between patch checks. Minimum `60`. Set to `0` to disable automatic checks. | `60` |
| `log-level` | string | Log verbosity. One of `debug`, `info`, `notice`, `warning`, `error`. | `warning` |
| `tls-patch-download` | boolean | Require TLS for all patch downloads. | `false` |
| `cutoff-date` <sup>1</sup> | string | RFC 3339 date in the past. Patches released after this date are not installed. | Empty |
| `patch-delay` <sup>1</sup> | string | Duration to wait before a newly released patch is applied. Allowed units: `s`, `m`, `h`, `d`, `w`. | `0` |

<sup>1</sup> Only available to paid Ubuntu Pro subscribers using Canonical's hosted Livepatch service (with the `remote-server` option unchanged).