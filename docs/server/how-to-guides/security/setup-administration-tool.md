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

## Authentication

The Livepatch administration tool authenticates with the Livepatch Server using Basic Auth (username and password).

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

livepatch-admin login --auth <username>:<password>
```

## Ubuntu SSO authentication (removed)

```{note}
Ubuntu SSO authentication has been removed. Use password (Basic Auth) authentication instead.
```

Ubuntu SSO authentication was previously supported but has been removed and is no longer available for on-premises deployments.