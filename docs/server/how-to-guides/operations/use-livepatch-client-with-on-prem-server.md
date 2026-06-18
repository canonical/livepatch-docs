---
myst:
  html_meta:
    description: "How to use a Livepatch Client with an on-premises server."
---

(server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server)=

# Use a Livepatch Client with an on-premises server

## Network access

Machines running the Livepatch Client require network access to the on-premises server. HTTPS (:443) or HTTP (:80) is used, depending on how the Livepatch on-premises haproxy and the Livepatch application's `url_template` setting are configured.

In addition, machines require access to the Canonical Snap Store to install the Livepatch Client snap:

* Snap Store: `api.snapcraft.io:443`
* Snap Store CDN: `*.snapcraftcontent.com:443`

## Generate the authorization token

To configure the Livepatch Client to pull patches from the on-premises Livepatch Server, an authorization token is required. To issue one, run:

```
livepatch-admin auth-token <id> <tier>
```

The tier parameter is one of the tiers available on the server. The client downloads patches as they become available in that tier. See [Patch management](/server/reference/patch-management/patch-management.md) for details on managing tiers and patches in the on-premises server.

The id parameter bears no significance in an on-premises deployment. It can be set to a value identifying the group of Livepatch Clients that will be enabled using the same token. A single authorization token can be used to enable multiple client instances.

## Configure the Livepatch Client

To start applying live kernel patches to a machine, install the Livepatch Client on it. The Livepatch Client is distributed as a snap. On the machine, run:

```
sudo snap install canonical-livepatch
```

After the client is installed, configure it to pull patches from the on-premises server:

```
canonical-livepatch config remote-server="http(s)://<hostname>"
```

The authorization token can then be used to attach any number of machines to the on-premises Livepatch Server:

```
canonical-livepatch enable <token>
```