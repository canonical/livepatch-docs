---
myst:
  html_meta:
    description: "Reference for machine reports in Livepatch on-prem, covering report configuration, generation, and querying via the livepatch-admin tool."
---

(server-reference-machine-reports)=

# Machine reports

Each machine running the Livepatch Client periodically checks in with its configured server. During this check-in, the machine queries for available patches and sends machine status information. For a full list of the information sent by clients, see the [Livepatch Client data reference](/client/reference/networking/data-sent.md).

## Enable machine reports

To enable machine reports and configure their retention period, see the [machine reports configuration options](/server/reference/platform/configuration.md) in the Livepatch Server configuration reference.

## Generate machine reports

Machine reports are generated using the Livepatch Admin Tool. See the [admin tool setup guide](/server/how-to-guides/security/setup-administration-tool.md) for instructions on configuring the tool.

Generate a machine report by querying machines by their tier, last applied patch version, or patch state:

```
livepatch-admin report machines <tier> [<patch version> [<patch state>]]
```