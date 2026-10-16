---
myst:
  html_meta:
    description: "How to setup administration tool with Livepatch on-prem."
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
Always run `livepatch-admin` commands with `sudo`. The tool writes files to the snap's common directory (for example, `$SNAP_COMMON/patches/`), and it fetches a resource token from the Livepatch client snap. Running commands without elevated privileges can cause permission errors.
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

The Livepatch administration tool authenticates with the Livepatch Server using Basic Auth (username and password).

```{note}
The `livepatch-admin` tool requires no cloud credentials or other cloud-specific setup. It authenticates with the server using only the administrator username and password via `livepatch-admin login`, regardless of the server's patch storage backend. For example, when uploading patches to a cloud-backed patch store, the server signs the temporary upload credentials itself.
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

Generate a username and password hash pair:

```
htpasswd -bnBC 10 <username> <password>
username:$2y$10$74ZpDgHaxnUQo.AJZk1cMuSRfef5oK5xq5o/GLbUH/Bbw6W2bmctm
```

Multiple pairs can be provided as a comma-separated list:

```
juju config livepatch auth_basic_users="<user1>:<password1,<user2>:<password2>"
```

When logging in with the client, provide the username and password:

```
export LIVEPATCH_URL={haproxy URL or unit IP}

sudo livepatch-admin login --auth <username>:<password>
```

## Ubuntu SSO authentication (removed)

```{note}
Ubuntu SSO authentication has been removed. Use password (Basic Auth) authentication instead.
```

Ubuntu SSO authentication was previously supported but has been removed and is no longer available for on-premises deployments.