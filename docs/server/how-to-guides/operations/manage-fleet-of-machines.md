---
myst:
  html_meta:
    description: "How to manage fleet of machines with Livepatch on-prem."
---

(server-how-to-guides-how-to-manage-livepatch-on-prem-fleet)=

# Manage a Livepatch on-premises fleet

The Livepatch server stores information about the status of machines attached to it. This information can be used to identify machines that failed to apply patches.

```
livepatch-admin report machines <tier> [<patch-version>] [<patch-state>]
```

The output of this command contains a list of machines, along with their machine IDs and additional information.

`<patch-state>` is one of:

* applied
* apply-failed
* unapplied
* needs-check
* nothing-to-apply
* unknown
* check-failed
* applied-with-bug
* Kernel-upgrade-required

The machine IDs correspond to unique Livepatch clients. To associate each client system with its machine ID, run the following command on the client:

```
cat /etc/machine-id
```