---
myst:
  html_meta:
    description: "Machine reports - learn about this topic in Livepatch on-prem."
---


(livepatch_on_prem-explanation-machine-reports)=

# Machine Reports

Each machine running the Livepatch Client will periodically check in with its configured server.

The machine checks for patches and sends machine status information. The full list of information sent is documented [here](/client/reference/data-sent.md).

## Enabling Machine Reports

See our [machine reports config](/server/reference/configuration.md) to enable machine reports and configure their retention period.

## Generating Machine Reports

To generate machine reports use the Livepatch admin tool.
See [our how-to](/server/how-to-guides/setup-administration-tool.md) on setting up the admin tool.

You can create machine reports where you query machines by their tier, last applied patch version or patch state.

`livepatch-admin report machines <tier> [<patch version> [<patch state]]`
