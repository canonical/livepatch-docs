---
myst:
  html_meta:
    description: "Register a Livepatch Client with your on-premises server using a local authorization token and configure its server address and certificate trust."
---

(server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server)=

# Use a Livepatch Client with an on-premises server

## Network access

Machines running the Livepatch Client require network access to the server's configured frontend and any separate patch-download endpoint. Client requests and reports go to that server, not directly to Canonical; server-to-upstream synchronization is described in the [data-exchange reference](/server/reference/telemetry/data-sent.md). Use HTTPS across untrusted networks and, for a private CA or self-signed certificate, configure {ref}`client certificate trust <configuring-livepatch-client-with-tls>` before registration. HTTP is optional within a fully trusted internal network.

In addition, machines require access to the Canonical Snap Store, or your approved local or offline snap-delivery process, to install the Livepatch Client snap:

* Snap Store: `api.snapcraft.io:443`
* Snap Store CDN: `*.snapcraftcontent.com:443`

## Generate the authorization token

To configure the Livepatch Client to pull patches from the on-premises Livepatch Server, an authorization token is required. To issue one, run:

```
sudo livepatch-admin --url 'http://<server-ip>' auth-token '<id>' '<tier>'
```

Run this from an administration host [logged in](/server/how-to-guides/security/setup-administration-tool.md) to your server. Replace `<server-ip>` with the IP address or hostname of your Livepatch frontend, and use `https://` if it terminates TLS.

The tier parameter is one of the tiers available on the server. The client downloads patches as they become available in that tier. See [Patch management](/server/reference/patch-management/patch-management.md) for details on managing tiers and patches in the on-premises server.

The `id` parameter is an operator-chosen value identifying the group of clients that will use this token; an Ubuntu SSO account is not required. A single authorization token can register multiple clients in its tier, so anyone who obtains it may be able to register additional clients. Distribute it through an approved secret-delivery mechanism and keep it out of public configuration, shared shell history, and provisioning logs. After registration, each client uses a machine token that your server validates locally.

## Configure the Livepatch Client

To start applying live kernel patches to a machine, install the Livepatch Client on it. The Livepatch Client is distributed as a snap. On the machine, run:

```
sudo snap install canonical-livepatch
```

After the client is installed, configure it to pull patches from the on-premises server:

```
sudo canonical-livepatch config remote-server='http://<server-ip>'
```

The authorization token can then be used to attach any number of machines to the on-premises Livepatch Server:

```
sudo canonical-livepatch enable '<authorization-token>'
```