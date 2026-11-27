---
myst:
  html_meta:
    description: "Register a Livepatch Client with your on-premises server using a local authorization token and configure its server address and certificate trust."
---

(server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server)=

# Use a Livepatch Client with an on-premises server

This guide uses an auth token issued by your Livepatch Server. The client registers with your server and receives a machine token for subsequent requests. Your deployment validates these credentials locally.

## Network access

Clients need network access to the server's configured frontend and any separate patch-download endpoint. Use HTTPS across untrusted networks. TLS terminates at the ingress or reverse proxy, not at the Livepatch Server itself. HTTP is optional within a fully trusted internal network.

For a private CA or self-signed frontend certificate, configure {ref}`client certificate trust <configuring-livepatch-client-with-tls>` before registration.

Installing and updating the client snap requires access to the Snap Store or your approved local or offline snap-delivery process. Direct online installation uses:

* Snap Store: `api.snapcraft.io:443`
* Snap Store CDN: `*.snapcraftcontent.com:443`

## Generate the authorization token

From an administration host authenticated to your on-premises server, request an authorization token:

```
sudo livepatch-admin --url 'http://<server-ip>' auth-token '<id>' '<tier>'
```

Replace `<server-ip>` with the IP address or hostname of your Livepatch frontend, and use `https://` if it terminates TLS. See [administration tool setup](/server/how-to-guides/security/setup-administration-tool.md) for login and credential configuration.

The tier parameter is one of the tiers available on the server. The client downloads patches as they become available in that tier. See [Patch management](/server/reference/patch-management/patch-management.md) for details on managing tiers and patches in the on-premises server.

Use an operator-chosen `id` identifying the group of clients that will use this token; an Ubuntu SSO account is not required. A single authorization token can register multiple clients in its assigned tier.

Distribute the token through an approved secret-delivery mechanism. Anyone who obtains it may be able to register additional clients. Keep tokens out of public configuration, shared shell history, and provisioning logs.

## Configure the Livepatch Client

On a machine with Snap Store access, install the client:

```
sudo snap install canonical-livepatch
```

For offline installations, use your approved local snap-delivery process instead.

Configure the client to use the on-premises frontend:

```
sudo canonical-livepatch config remote-server='http://<server-ip>'
```

Register the machine with the authorization token:

```
sudo canonical-livepatch enable '<authorization-token>'
```

After registration, the client uses its machine token for subsequent requests. Protect privileged access to the client host and its stored authentication material. Client requests and reports go to your configured on-premises server; separate server-to-upstream synchronization is described in the [data-exchange reference](/server/reference/telemetry/data-sent.md).