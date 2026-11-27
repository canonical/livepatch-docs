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

## Authentication

For normal administration of your on-premises server, the tool uses the Basic Auth username and password configured on that server. This login does not require the Ubuntu Pro client or a connection to Canonical's hosted service.

```{note}
Administrator login does not require cloud-storage credentials, regardless of the server's patch storage backend. Storage credentials belong in the server deployment's configuration, not the administration tool's login.
```

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

Multiple pairs can be provided as a comma-separated list of hashes, not plaintext passwords:

```
<user1>:<bcrypt-hash1>,<user2>:<bcrypt-hash2>
```

For the current machine and Kubernetes charms, put this list in a Juju secret and configure `auth.basic.enabled=true` with `auth.basic.users-secret`. Follow the [secret-backed administrator configuration procedure](/server/how-to-guides/security/harden-your-deployment.md#secure-admin-credentials).

For a standalone snap, configure the list directly:

```shell
sudo snap set canonical-livepatch-server lp.auth.basic.enabled=true lp.auth.basic.users='<user1>:<bcrypt-hash1>,<user2>:<bcrypt-hash2>'
```

Treat password hashes as sensitive and protect configuration and any exported copies. Do not manually change the server snap's settings when it is managed by a charm.

Use individual administrator accounts. When setting the username-and-hash list, include every account that should remain configured: setting this option replaces the entire list.

When logging in with the administration tool, provide the username and original password, not the hash. Use `https://` when connecting across an untrusted network, and configure [certificate trust](/server/how-to-guides/security/setup-tls.md#configure-the-livepatch-admin-tool-with-tls) if needed:

```
sudo livepatch-admin --url 'http://<server-ip>' login --auth '<username>:<password>'
```

Replace `<server-ip>` with the IP address or hostname of your Livepatch frontend, such as the HAProxy unit or ingress. Use the same `--url` on subsequent administration commands to select the intended server.

Run login commands on a trusted administration host. Password arguments can be exposed through shell history and process listings; do not save populated login commands in scripts or shared history.

## Change administrator credentials

Generate a replacement bcrypt hash interactively as described above. Update the complete username-and-hash list, preserving the other accounts that should remain configured. Protect temporary files and avoid exposing hashes in shared command history or configuration exports.

### Machine and Kubernetes charms

If you use `auth.basic.users` directly, first move the list to a Juju secret using the [secret-backed configuration procedure](/server/how-to-guides/security/harden-your-deployment.md#secure-admin-credentials).

For an existing secret-backed configuration, identify the configured secret:

```shell
juju config livepatch auth.basic.users-secret
```

Prepare an updated `admin-users.secret.yaml` file with permissions `0600`, outside version control. Its `value` key must contain the complete updated username-and-hash list. Update the existing secret using the URI returned above:

```shell
juju update-secret '<secret-uri>' --file admin-users.secret.yaml
```

The examples assume that the application is named `livepatch`. Use your deployment's application name and existing secret URI. Remove the temporary file when finished, retaining any required copy only in an approved credential store.

### Standalone snap

Update the complete list through snap configuration:

```shell
sudo snap set canonical-livepatch-server lp.auth.basic.users='<complete-username-and-bcrypt-hash-list>'
```

For either deployment method, allow configuration changes to complete and confirm that a fresh administrator login succeeds with the updated credentials.

## Downloading patches from Canonical's hosted server

This additional setup is only for downloading patches from Canonical's hosted Livepatch Server, for example to transfer patches across an air gap. On the connected system running the admin tool, enable the Livepatch client:

```
sudo pro attach <ubuntu-pro-token>
sudo pro enable livepatch
```

The Ubuntu Pro token must be entitled to Livepatch. Attaching the subscription and enabling Livepatch provides the resource token that the administration tool uses when downloading from the hosted service.

```{note}
Downloading patches from Canonical's hosted service with the admin tool requires a paid Ubuntu Pro subscription. Free-tier tokens do not allow these downloads. This requirement does not apply to ordinary administrator login to your own server.
```

## Ubuntu SSO authentication (removed)

```{note}
Ubuntu SSO authentication has been removed. Use password (Basic Auth) authentication instead.
```

Ubuntu SSO authentication was previously supported but has been removed and is no longer available for on-premises deployments.