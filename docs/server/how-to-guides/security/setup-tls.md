---
myst:
  html_meta:
    description: "How to setup tls with Livepatch on-prem."
---

(server-how-to-guides-how-to-setup-tls-for-livepatch-on-prem)=

# Set up TLS for Livepatch on-premises

The security of live kernel patching depends not only on the signed kernel modules but also on the secure TLS channel between the Livepatch Client and the on-premises server. Setting up the necessary TLS keys and certificates for the on-premises service is therefore essential.

There are several ways to set up TLS for Livepatch on-premises. One approach is to use a dedicated TLS-terminating reverse proxy in front of the haproxy service. Another approach is to configure the appropriate TLS certificate and key on the haproxy instance directly.

## Configure TLS for haproxy

The following steps configure TLS on the haproxy service directly:

1. Create a `tls-overlay.yaml` file with the following content:

```
applications:
  haproxy:
    options:
      services: |
        - service_name: livepatch
          service_host: "0.0.0.0"
          service_port: 443
          service_options:
            - balance leastconn
            - cookie SRVNAME insert
            - 'acl restricted_api path_beg,url_dec -i /api/auth-tokens'
            - http-request deny if restricted_api
          server_options: maxconn 200 cookie S{i} check
          crts: [DEFAULT]
      ssl_cert: include-base64://./cert.pem
      ssl_key: include-base64://./key.pem
```

2. Rename the certificate and key files to `cert.pem` and `key.pem` and place them in the same directory as the downloaded overlay file.
3. Run the following Juju command:

```
juju deploy ch:canonical-livepatch-onprem --overlay ./tls-overlay.yaml
```

4. Run `juju status` to verify that the haproxy service is now exposing port 443.

![screenshot_20210603_165758|690x55](/_static/images/nuS5aGmo44qwv5vagLczO0tnfog.png)

(server-how-to-guides-configuring-livepatch-admin-tool-with-tls)=

## Configure the Livepatch admin tool with TLS

If the TLS certificate used for Livepatch originates from a trusted CA, no further configuration is required. The Livepatch admin tool uses the configured system certificates to verify the server's responses.

If a self-signed certificate is used, the administration tool must be configured to use the certificate. There are several ways to do this.

### Command line option

The command line option for the Livepatch admin tool accepts either the path to a certificate chain PEM file, or the contents of the certificate chain:

```
livepatch-admin --ca ./cert.pem login -a (...)
```

```
livepatch-admin --ca "$(cat ./cert.pem)" login -a (...)
```

### Environment variable

The environment variable `LIVEPATCH_CA_CRT` can be set with either the path to a certificate chain file or the contents of the certificate chain:

```
export LIVEPATCH_CA_CRT='/temp/cert.pem'
```

```
export LIVEPATCH_CA_CRT="$(cat ./cert.pem)"
```

## Configure the Livepatch Client with TLS

If a self-signed certificate is used for the Livepatch on-premises service, Livepatch Client instances must also be configured with that certificate to verify responses from the on-premises server:

```
sudo canonical-livepatch config ca-certs=@stdin < ./cert.pem
```

For more information on configuring an HTTPS proxy on the Livepatch Client, see the [proxy configuration guide](/client/how-to-guides/configuration/configure-proxy.md).