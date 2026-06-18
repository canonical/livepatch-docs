---
myst:
  html_meta:
    description: "How to fetch patches with Livepatch on-prem."
---

(server-how-to-guides-how-to-fetch-patches-to-livepatch-on-prem-server)=

# How to fetch patches to livepatch on-prem server

The livepatch application is configured to fetch patch updates every 24 hours. This setting is [configurable](/server/reference/platform/configuration.md).

Patch snapshot downloads can also be manually triggered:

```
livepatch-admin sync trigger --wait
```

It is recommended to trigger a patch snapshot download once the server is successfully set up.

## Verifying that server is up to date

To verify that the server is receiving the latest patches from the livepatch server hosted by Canonical use the following command.

```
livepatch-admin sync reports
```
