---
myst:
  html_meta:
    description: "How to securely configure and operate the client with Livepatch client."
---


(client-how-to-guides-how-to-securely-configure-and-operate-the-livepatch-client)=

# How to securely configure and operate the Livepatch client

The Livepatch client is available for use as a confined snap, and therefore only has access to the specific interfaces defined for it. This massively reduces the attack vector, in the case that the Livepatch client is compromised. However, administrators and users of the Livepatch client snap must still ensure that the recommended security practices are followed, to maintain the security of the snap in their environments.

## Use the latest version

The Livepatch client snap must always be installed from the [latest/stable channel](https://snapcraft.io/canonical-livepatch). This channel serves the latest stable release of the snap, and all security updates will be made available through this channel.

Use the following commands to check if the latest version of the Livepatch client snap has been installed in the system.

```shell
# Install yq if not already present
sudo snap install yq
# Check the version of canonical-livepatch in the latest/stable channel
snap info canonical-livepatch | yq '.channels.latest/stable' | awk '{print $1}'
# Check the version of canonical-livepatch installed in the system
snap info canonical-livepatch | yq '.installed' | awk '{print $1}'
```

If the versions do not match, update the Livepatch client snap to the latest version by running the following command.

```
sudo snap refresh canonical-livepatch --channel=latest/stable
```

## Encrypt connection between the client and the server

The default configuration of the Livepatch client uses the Canonical-hosted Livepatch server as the remote-server, with no additional ca-certs or proxy configurations enabled. Using the default configuration ensures that the data is always encrypted in transit, as the Livepatch client uses TLS (minimum v1.2) with server-side authentication while communicating with the Canonical-hosted Livepatch server.

However, for on-premises Livepatch server deployments, users can decide to forgo TLS while communicating with the Livepatch server, as there is no strict TLS enforcement at the client-side. It is strongly recommended to enable TLS for communication between the client and the on-prem Livepatch servers, to prevent attacks from malicious actors. The Livepatch client supports using TLS for such communications, by providing a way to configure additional CA certificates and route traffic through proxies. See these how-to guides on [configuring the client for TLS with custom certificates](/server/how-to-guides/setup-tls.md#configuring-livepatch-client-with-tls) and [configuring the client to use proxies](/client/how-to-guides/configure-proxy.md).

It is also recommended to enforce TLS for patch downloads, when using an on-premises deployment of the Livepatch server, by setting the `tls-patch-download` configuration option to `true`.

## Maintain strict privileged access

The Livepatch client snap does not encrypt its data at rest. However, all client specific data is stored in the [$SNAP_COMMON](https://snapcraft.io/docs/reference/development/environment-variables/#snap-common) and [$SNAP_DATA](https://snapcraft.io/docs/reference/development/environment-variables/#snap-data) directories. For further information, see the [security overview for snaps](https://snapcraft.io/docs/explanation/security/security-policies/#security-overview) and the documentation on [data locations used by snaps](https://snapcraft.io/docs/reference/administration/data-locations/#data-locations).

By default, the Livepatch client runs as a privileged daemon process and stores all necessary data in these directories. The stored data can only be accessed by the root user for reads and writes. Therefore, maintaining strict privileged access controls on machines running the Livepatch client snap, is essential to preventing unauthorized access of the snap data.

## Configuring the Livepatch client

See these how-to guides and configuration references to understand how to configure the Livepatch client and the configuration options available:

1. [How to configure the Livepatch client](/client/how-to-guides/configure-livepatch-client.md)
2. [Livepatch client configuration options reference](/client/reference/configuration-options.md)
