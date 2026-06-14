---
myst:
  html_meta:
    description: "Configuration Options - technical reference for Livepatch client."
---


(livepatch-reference-livepatch-client-configuration-options)=

# Livepatch client configuration options

The following configuration options are available for the Livepatch client, which can be set following the [How to configure Livepatch client](/client/how-to-guides/configure-livepatch-client.md) how-to guide.

|Key | Data Type | Description | Default Value|
|--- | --- | --- | ---|
|`http-proxy` | string | Value passed as `HTTP_PROXY` (overrides `/etc/environment`) | Empty|
|`https-proxy` | string | Value passed as `HTTPS_PROXY` (overrides `/etc/environment`) | Empty|
|`no-proxy` | string | Value passed as `NO_PROXY` (overrides `/etc/environment`) | Empty|
|`remote-server` | string | Livepatch server URL | `https://livepatch.canonical.com`|
|`ca-certs` | string | Custom CA root certificate(s) encoded as PEM | Empty|
|`dial-timeout` | string | Timeout for opening TCP connections; allowed units are `s`, `m`, `h` | `12s`|
|`check-interval` | integer | Minutes between checks for new patches. Minimum `60`. Use `0` to disable auto refresh | `60`|
|`log-level` | string | One of `debug`, `info`, `notice`, `warning`, `error` | `warning`|
|`tls-patch-download` | boolean | Enforce using TLS for patch downloads | `false` |
|`cutoff-date` <sup>1</sup> | string | RFC3339 date in the past after which new patched will not be installed | Empty|
|`patch-delay` <sup>1</sup> | string | Duration before a newly released patch is received by the client; allowed units are `s`, `m`, `h`, `d`, `w` | `0`|

<sup>1</sup> Only available to paid Ubuntu Pro users who are using Canonical-hosted Livepatch, with the `remote-server` option unchanged
