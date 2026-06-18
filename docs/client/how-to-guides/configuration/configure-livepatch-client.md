---
myst:
  html_meta:
    description: "How to configure the Livepatch Client."
---


(client-how-to-guides-how-to-configure-the-livepatch-client)=

# How to configure the Livepatch Client

The Livepatch Client can be configured using the CLI or its configuration file at `/var/snap/canonical-livepatch/common/config`.

## CLI configuration

Show the current configuration:

```
canonical-livepatch config
```

Change one or more settings:

```
canonical-livepatch config http-proxy="1.2.3.4" https-proxy="1.2.3.4"
canonical-livepatch config remote-server="https://example.livepatch.canonical.com"
```

Clear one or more settings:

```
canonical-livepatch config remote-server=
```

Change settings, reading a long, multi-line value from stdin:

```
canonical-livepatch config remote-server=https://2.3.4.5 ca-certs=@stdin < chain.pem
```

## YAML configuration

The Livepatch Client can also be configured by editing `/var/snap/canonical-livepatch/common/config`. The file is YAML-formatted. For changes to the file to take effect, restart the daemon:

```
sudo snap restart canonical-livepatch
```
