---
myst:
  html_meta:
    description: "Configure a TLS frontend for a charmed or standalone Livepatch Server and configure certificate trust for administrators and clients."
---

(server-how-to-guides-how-to-setup-tls-for-livepatch-on-prem)=

# Set up TLS for Livepatch on-premises

The Livepatch Server serves HTTP, so TLS is terminated by a frontend such as an ingress or reverse proxy. TLS is optional within a fully trusted internal network, but recommended when clients or administrators connect across untrusted networks. Obtain a certificate for a hostname that clients and administrators can resolve and configure it on the frontend using the provider's documentation. Protect the private key, arrange renewal, and restrict direct access to the HTTP backend so that clients cannot bypass the frontend. The connection from the frontend to Livepatch is not encrypted, so keep it on a trusted network or protect it at the network layer. The Juju examples assume an application named `livepatch`. Replace `<server-ip>` with the hostname or IP address of the frontend, matching a name on its TLS certificate.

## Configure a Kubernetes charm frontend

For new deployments, use the `ingress` integration with a provider such as [Traefik K8s](https://charmhub.io/traefik-k8s) or [Gateway API Integrator](https://charmhub.io/gateway-api-integrator). Existing deployments can use the `nginx-route` integration with [Nginx Ingress Integrator](https://charmhub.io/nginx-ingress-integrator) and `ingress-interface=legacy-nginx-route`, which remains the default; prepare the replacement provider before switching. To connect an existing Traefik application named `traefik-k8s`:

```
juju config livepatch ingress-interface=ingress
juju integrate livepatch:ingress traefik-k8s
```

(configure-tls-for-haproxy)=

## Configure a machine charm frontend

Use [HAProxy](https://charmhub.io/haproxy) as the frontend and configure its HTTPS listener and certificate using the HAProxy charm's documentation. The relation below connects only the HTTP backend and does not by itself configure a TLS certificate. For an application named `haproxy`:

```
juju integrate livepatch:website haproxy:reverseproxy
```

## Configure a standalone snap frontend

Configure a reverse proxy such as Nginx or HAProxy to accept HTTPS and forward requests to the address in `lp.server.server-address`, which defaults to `localhost:8080`. If the proxy runs on another host, bind the backend to a private address and restrict access to it.

## Configure client-facing addresses

If patch downloads are served through the frontend, configure the patch URL template to use its HTTPS address and keep `/v1/patches/{filename}` literal. A separate patch-download service needs its own HTTPS and certificate trust. For either charm:

```
juju config livepatch server.url-template='https://<server-ip>/v1/patches/{filename}'
```

For the standalone snap:

```
sudo snap set canonical-livepatch-server lp.server.url-template='https://<server-ip>/v1/patches/{filename}'
```

(server-how-to-guides-configuring-livepatch-admin-tool-with-tls)=

## Configure the Livepatch admin tool with TLS

If the TLS certificate used for Livepatch originates from a trusted CA, no further configuration is required. The Livepatch admin tool uses the configured system certificates to verify the server's responses.

For a private CA or self-signed certificate, supply the trust certificate instead of disabling certificate verification. When a CA is supplied with either method below, the admin tool requires TLS 1.3, so the frontend must accept TLS 1.3 connections.

### Command line option

The command line option for the Livepatch admin tool accepts either the path to a certificate chain PEM file, or the contents of the certificate chain:

```
sudo livepatch-admin --url 'https://<server-ip>' --ca ./ca-chain.pem login --auth '<username>:<password>'
```

### Environment variable

Alternatively, set `LIVEPATCH_CA_CRT` to the certificate file's absolute path or its PEM contents. With `sudo`, set the variable for the privileged process:

```
sudo env LIVEPATCH_CA_CRT=/absolute/path/ca-chain.pem livepatch-admin --url 'https://<server-ip>' login --auth '<username>:<password>'
```

(configuring-livepatch-client-with-tls)=

## Configure the Livepatch Client with TLS

Configure clients to use the HTTPS frontend address following the [client setup guide](/server/how-to-guides/operations/use-livepatch-client-with-on-prem-server.md). For a private CA or self-signed certificate, also configure the trust certificate on each client:

```
sudo canonical-livepatch config ca-certs=@stdin < ./ca-chain.pem
```

For more information on configuring an HTTPS proxy on the Livepatch Client, see the [proxy configuration guide](/client/how-to-guides/configuration/configure-proxy.md).