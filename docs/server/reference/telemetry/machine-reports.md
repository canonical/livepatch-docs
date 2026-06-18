---
myst:
  html_meta:
    description: "Machine reports - learn about this topic in Livepatch on-prem."
---


(server-reference-machine-reports)=

# Machine Reports

Each machine running the Livepatch Client will periodically check in with its configured server.

The machine checks for patches and sends machine status information. The full list of information sent is documented [here](/client/reference/networking/data-sent.md).

## Enabling Machine Reports

See the [machine reports config](/server/reference/platform/configuration.md) to enable machine reports and configure their retention period.

## Generating Machine Reports

To generate machine reports use the Livepatch admin tool.
See [the how-to](/server/how-to-guides/security/setup-administration-tool.md) on setting up the admin tool.

You can create machine reports where you query machines by their tier, last applied patch version or patch state.

`livepatch-admin report machines <tier> [<patch version> [<patch state]]`
