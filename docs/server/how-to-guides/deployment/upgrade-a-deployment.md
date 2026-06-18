---
myst:
  html_meta:
    description: "How to upgrade a Livepatch on-premises deployment."
---


(server-how-to-guides-how-to-upgrade-a-livepatch-on-prem-deployment)=

# How to upgrade a deployment

To upgrade the Livepatch on-premises deployment, each application must be upgraded separately:

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

After upgrading the Livepatch application, a schema upgrade may be required. This is indicated in the application's status when running `juju status`. In such a case, run:

```
juju run-action livepatch/leader schema-upgrade --wait
```

The Livepatch application can also be restarted with the following command:

```
juju run-action livepatch/<desired-unit> restart --wait
```

> `juju run-action` has been renamed to `juju run` from Juju v3.
> See the [Juju documentation on managing actions](https://documentation.ubuntu.com/juju/latest/howto/manage-actions/#run-an-action) for more details.
