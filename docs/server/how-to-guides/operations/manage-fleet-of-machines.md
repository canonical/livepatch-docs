---
myst:
  html_meta:
    description: "How to manage fleet of machines with Livepatch on-prem."
---

(server-how-to-guides-how-to-manage-livepatch-on-prem-fleet)=

# How to manage Livepatch on-prem fleet

Livepatch server stores information about the status of machines attached to it. This can be used to identify machines which failed to apply patches.

```
livepatch-admin report machines <tier> [<patch-version>] [<patch-state>]
```

The output of this command will contain a list of machines, along with their machine IDs and additional information.

<patch-state> is one of:

- applied
- apply-failed
- unapplied
- needs-check
- nothing-to-apply
- unknown
- check-failed
- applied-with-bug
- Kernel-upgrade-required

Note that the machine IDs correspond to unique Livepatch clients. To associate each client system with the machine ID you can run the command below on the client.

```
cat /etc/machine-id
```
