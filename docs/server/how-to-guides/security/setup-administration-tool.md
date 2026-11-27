---
myst:
  html_meta:
    description: "Install the Livepatch administration tool, configure local administrator authentication, and change administrator credentials securely."
---

(server-how-to-guides-how-to-setup-livepatch-on-prem-server-administration-tool)=

# Set up the Livepatch on-premises server administration tool

To perform operations such as promoting patches to tiers and issuing tokens for machines to attach to the Livepatch Server, an administration tool is provided as a snap:

```
sudo snap install canonical-livepatch-server-admin
```

For ease of use, alias the admin command:

```
sudo snap alias canonical-livepatch-server-admin.livepatch-admin livepatch-admin
```

```{note}
Run administration commands with `sudo` as shown in this guide. The tool writes to protected snap storage, including downloaded patch files. Only operations that require access to Canonical's hosted service need the additional Ubuntu Pro setup described below.
```

## Downloading patches from Canonical's hosted server

To download patches from Canonical's hosted Livepatch Server (for example, to sync patches for an on-premises deployment across an air gap), the system running the admin tool must have the Livepatch client enabled:

```
sudo pro attach <ubuntu-pro-token>
sudo pro enable livepatch
```

The Ubuntu Pro token must be entitled to Livepatch. Attaching the subscription and enabling Livepatch on the VM provides the resource token the admin tool fetches from the client snap when downloading patches.

```{note}
Downloading patches with the admin tool requires a paid Ubuntu Pro subscription. Free-tier tokens do not allow downloading patches.
```

## Authentication

For normal administration of your on-premises server, the tool uses the Basic Auth username and password configured on that server. This login does not require the Ubuntu Pro client, a connection to Canonical's hosted service, or cloud-storage credentials, regardless of the server's patch storage backend.

(server-how-to-guides-password-authentication)=

## Password authentication

To configure password authentication, username and password hash pairs must be generated using the `htpasswd` tool available in the `apache2-utils` package.

````{note}
The `apache2-utils` package can be installed using the following commands:

```
sudo apt-get update
sudo apt-get install apache2-utils
```
````

Generate a username and bcrypt password hash pair. Enter the password when prompted, rather than passing it as a command argument:

```
htpasswd -nBC 10 '<username>'
username:$2y$10$74ZpDgHaxnUQo.AJZk1cMuSRfef5oK5xq5o/GLbUH/Bbw6W2bmctm
```

Use individual administrator accounts. Multiple pairs can be provided as a comma-separated list of hashes, not plaintext passwords. Setting the list replaces the entire list, so include every account that should remain configured. For the machine and Kubernetes charms, put this list in a Juju secret and configure `auth.basic.enabled=true` with `auth.basic.users-secret` by following the [secret-backed administrator configuration procedure](/server/how-to-guides/security/harden-your-deployment.md#secure-admin-credentials). Do not manually change the server snap's settings when it is managed by a charm. For a standalone snap, configure the list directly:

```
sudo snap set canonical-livepatch-server lp.auth.basic.enabled=true lp.auth.basic.users='<user1>:<bcrypt-hash1>,<user2>:<bcrypt-hash2>'
```

When logging in with the administration tool, provide the username and original password, not the hash:

```
sudo livepatch-admin --url 'http://<server-ip>' login --auth '<username>:<password>'
```

Replace `<server-ip>` with the IP address or hostname of your Livepatch frontend, such as the HAProxy unit or ingress, and use the same `--url` on later commands. Use `https://` across untrusted networks and configure [certificate trust](/server/how-to-guides/security/setup-tls.md#configure-the-livepatch-admin-tool-with-tls) if needed. Password arguments can be exposed through shell history and process listings, so do not save populated login commands in scripts or shared history.

## Change administrator credentials

Generate a replacement hash as described above and update the complete username-and-hash list, preserving any other accounts. For the machine and Kubernetes charms, if you set `auth.basic.users` directly, first move the list to a Juju secret as described above. Then find the configured secret URI, prepare an updated `admin-users.secret.yaml` file whose `value` key contains the complete list, and update the secret:

```
juju config livepatch auth.basic.users-secret
juju update-secret '<secret-uri>' --file admin-users.secret.yaml
```

Keep the file outside version control with permissions `0600`, and remove it when finished. For the standalone snap, run the `snap set` command above with the complete list. In either case, confirm that a fresh administrator login succeeds with the updated credentials.

## Ubuntu SSO authentication (removed)

```{note}
Ubuntu SSO authentication has been removed. Use password (Basic Auth) authentication instead.
```

Ubuntu SSO authentication was previously supported but has been removed and is no longer available for on-premises deployments.