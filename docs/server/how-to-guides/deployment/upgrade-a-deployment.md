---
myst:
  html_meta:
    description: "How to upgrade a deployment with Livepatch on-prem."
---

(server-how-to-guides-how-to-upgrade-a-livepatch-on-prem-deployment)=

# How to upgrade a Livepatch on-prem deployment

To upgrade the livepatch on-prem deployment, each application needs to be upgraded separately:

```
juju refresh haproxy
```

```
juju refresh postgresql
```

```
juju refresh ubuntu-advantage
```

```
juju refresh livepatch
```

After upgrading the livepatch application, a schema upgrade may be required. This will be indicated in the application's status when running `juju status`. In such a case, run the command:

```
juju run-action livepatch/leader schema-upgrade --wait
```

One can also restart the livepatch application with the following command:

```
juju run-action livepatch/<desired-unit> restart --wait
```

> Note: `juju run-action` has been renamed to `juju run` from Juju v3
> See https://documentation.ubuntu.com/juju/latest/howto/manage-actions/#run-an-action for more details.
