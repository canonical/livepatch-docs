---
myst:
  html_meta:
    description: "How to fetch patches with Livepatch on-prem."
---

(server-how-to-guides-how-to-fetch-patches-to-livepatch-on-prem-server)=

# Fetch patches to the Livepatch on-premises server

The Livepatch application is configured to fetch patch updates every 24 hours. This setting is [configurable](/server/reference/platform/configuration.md).

Patch snapshot downloads can also be manually triggered:

```
livepatch-admin sync trigger --wait
```

It is recommended to trigger a patch snapshot download once the server is successfully set up.

## Verify that the server is up to date

To verify that the server is receiving the latest patches from the Livepatch server hosted by Canonical, use the following command.

```
livepatch-admin sync reports
```