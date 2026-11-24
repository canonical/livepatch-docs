---
myst:
  html_meta:
    description: "Configure a TLS frontend for a charmed or standalone Livepatch Server and configure certificate trust for administrators and clients."
---

(server-how-to-guides-how-to-setup-tls-for-livepatch-on-prem)=

# Set up TLS for Livepatch on-premises

Use this guide to place a TLS-terminating frontend in front of an existing Livepatch Server. The server itself serves HTTP. TLS is optional within a fully trusted internal network, but recommended when clients or administrators connect across untrusted networks.

The Juju examples assume that your Livepatch application is named `livepatch`. Replace application names with your deployment's values. Use `<hosted url>` for your deployment's complete HTTPS frontend URL, without a trailing slash.

## Prepare the frontend

Choose a hostname that clients and administrators can resolve. Obtain a certificate for that hostname and configure its certificate chain and private key on the frontend using the frontend provider's supported procedure. Protect the private key and arrange certificate renewal.

For a private CA, obtain the issuing CA certificate chain for distribution to clients and administrators. Distribute certificates, not the provider's private key.

Restrict administrator endpoints to trusted administration networks. Restrict direct access to the HTTP backend so that clients cannot bypass frontend controls. TLS on the public frontend does not encrypt its HTTP connection to Livepatch; keep that connection on a trusted network or protect it at the network layer.

## Connect the frontend

### Kubernetes charm

For new deployments, use the `ingress` integration with a supported provider such as [Traefik K8s](https://charmhub.io/traefik-k8s) or [Gateway API Integrator](https://charmhub.io/gateway-api-integrator). Configure the provider's hostname, TLS certificate, and HTTPS listener according to its documentation.

For an existing Traefik application named `traefik-k8s`, connect Livepatch with:

```shell
juju config livepatch ingress-interface=ingress
juju integrate livepatch:ingress traefik-k8s
```

Existing deployments can use the `nginx-route` integration with [Nginx Ingress Integrator](https://charmhub.io/nginx-ingress-integrator) and `ingress-interface=legacy-nginx-route`. That selector remains the default for backward compatibility. Do not switch an existing deployment's interface without first preparing the replacement provider and planning the traffic change.

(configure-tls-for-haproxy)=

### Machine charm

Use [HAProxy](https://charmhub.io/haproxy) as the frontend. Deploy or select an HAProxy application and configure its HTTPS listener, hostname, certificate chain, and private key using the HAProxy charm's documentation.

For an application named `haproxy`, connect its `website` endpoint to Livepatch:

```shell
juju integrate livepatch:website haproxy:website
```

The relation connects the HTTP backend; it does not by itself configure a TLS certificate or guarantee an HTTPS listener. Confirm that the frontend serves HTTPS before directing clients to it.

### Standalone snap

Configure a reverse proxy or load balancer such as Nginx or HAProxy to accept HTTPS and forward requests to the address configured in `lp.server.server-address`. The standalone snap defaults to `localhost:8080`; retain a loopback backend when the proxy runs on the same host.

If the proxy runs on another host, bind the backend to an appropriate private address and restrict access to the proxy and other authorized callers. Configure certificates, renewal, access controls, and timeouts using your proxy's documentation.

## Configure client-facing addresses

Use the HTTPS frontend address for client registration and administration. If patch downloads are served through this frontend, configure the patch URL template to use it too.

Use `<hosted-url>` for the externally reachable patch-download base URL, including its scheme and any required path prefix, without a trailing slash. Keep `{filename}` literal.

For either charm:

```shell
juju config livepatch server.url-template='<hosted-url>/{filename}'
```

For the standalone snap:

```shell
sudo snap set canonical-livepatch-server lp.server.url-template='<hosted-url>/{filename}'
```

If your deployment uses a separate patch-download service, retain its intended URL and configure HTTPS and certificate trust for that service separately.

(server-how-to-guides-configuring-livepatch-admin-tool-with-tls)=

## Configure the Livepatch admin tool with TLS

If the frontend certificate chains to a CA trusted by the administration tool's system, no additional CA configuration is required. Use the HTTPS URL when [logging in](/server/how-to-guides/security/setup-administration-tool.md#password-authentication).

For a private CA or self-signed certificate, supply the appropriate trust certificate. Do not disable certificate verification to bypass a trust or hostname error.

### Command line option

The `--ca` option accepts a path to a PEM certificate chain file or its PEM contents. For example:

```shell
sudo livepatch-admin --url '<hosted url>' --ca ./ca-chain.pem login --auth '<username>:<password>'
```

### Environment variable

Alternatively, set `LIVEPATCH_CA_CRT` to the certificate file's absolute path or its PEM contents. When using `sudo`, set the variable for the privileged process rather than relying on the unprivileged shell's environment:

```shell
sudo env LIVEPATCH_CA_CRT=/absolute/path/ca-chain.pem \
    livepatch-admin --url '<hosted url>' login --auth '<username>:<password>'
```

Run login commands on a trusted administration host. Password arguments can be exposed through shell history and process listings; do not save populated login commands in scripts or shared history.

(configuring-livepatch-client-with-tls)=

## Configure the Livepatch Client with TLS

Configure clients to use the HTTPS frontend address following the [client setup guide](/server/how-to-guides/operations/use-livepatch-client-with-on-prem-server.md).

For a private CA or self-signed certificate, configure the trust certificate on each client:

```shell
sudo canonical-livepatch config ca-certs=@stdin < ./ca-chain.pem
```

After configuration, verify that an administrator can authenticate and a registered client can check for and download patches through the HTTPS frontend with certificate verification enabled. Confirm that direct backend access is restricted as intended.

If clients also use an outbound HTTPS proxy, see the [proxy configuration guide](/client/how-to-guides/configuration/configure-proxy.md).