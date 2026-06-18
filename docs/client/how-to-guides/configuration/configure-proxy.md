---
myst:
  html_meta:
    description: "How to configure proxy with Livepatch Client."
---


(client-how-to-guides-how-to-configure-a-proxy-in-the-livepatch-client)=

# How to configure a proxy in the Livepatch Client

The Livepatch Client supports communicating with the Livepatch Server through HTTP, HTTPS, or SOCKS5 proxies. Several configuration parameters control this behaviour.

## Check proxy configuration

To check the proxy configuration of the Livepatch Client, run the following command:

```bash
canonical-livepatch config
http-proxy: "http://proxy.example.com"
https-proxy: "http://proxy.example.com"
no-proxy: ""
ca-certs: ""
...
```

An empty string value (`""`) means the corresponding parameter is not set and system defaults are used.

## Use an HTTP proxy or SOCKS5 proxy

To enable an HTTP proxy, run the following commands:

```bash
sudo canonical-livepatch config http-proxy=http://proxy.example.com
sudo canonical-livepatch config https-proxy=http://proxy.example.com
```

To configure a SOCKS5 proxy, run these commands:

```bash
sudo canonical-livepatch config http-proxy=socks5://proxy.example.com:1080
sudo canonical-livepatch config https-proxy=socks5://proxy.example.com:1080
```

The client respects the standard Linux environment variables for proxy setup (`HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`). However, these variables must be set in the Livepatch Client daemon process environment to take effect. Using the configuration parameters above is the recommended approach.

## Use an HTTPS proxy

When using an HTTPS proxy (not to be confused with proxying HTTPS requests), include the `https://` scheme when setting the configuration parameters:

```bash
sudo canonical-livepatch config http-proxy=https://proxy.example.com
sudo canonical-livepatch config https-proxy=https://proxy.example.com
```

If a self-signed CA certificate is included in the HTTPS proxy's TLS certificate chain, add the CA certificate to the trusted certificates on the host machine. Run the following commands (assuming `ca.crt` is the CA certificate file):

```bash
sudo apt-get install ca-certificates
sudo cp ca.crt /usr/share/ca-certificates
sudo dpkg-reconfigure ca-certificates
```

To trust a self-signed CA certificate without installing it system-wide, explicitly instruct the Livepatch Client to trust the CA certificate:

```bash
sudo canonical-livepatch config ca-certs=@stdin < ca.crt
```

## Route directly to the Livepatch Server

If a system-wide proxy is already configured (for example, through the `HTTP_PROXY` environment variable), bypass it for communication with the Livepatch Server using the following configuration:

```bash
sudo canonical-livepatch config no-proxy=canonical.com
```
