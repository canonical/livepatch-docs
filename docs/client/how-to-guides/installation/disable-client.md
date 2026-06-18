---
myst:
  html_meta:
    description: "How to disable client with Livepatch Client."
---


(client-how-to-guides-how-to-disable-the-livepatch-client)=

# How to disable the Livepatch Client

Several methods are available to disable the Livepatch Client.

When direct access to the system is available, disable the Livepatch service:

```
sudo snap stop --disable canonical-livepatch
```

When direct access to the system is not available, the Livepatch Client can be disabled in two ways:

* Set a kernel command line parameter `canonical_livepatch_mode`
* Write the mode to the `/var/local/canonical_livepatch_mode` file

These two locations are checked only when the Livepatch daemon is started, typically at boot.

The mode value can be:

* `normal`: The default operation mode. Patch information is refreshed regularly and new patches are applied.
* `no-apply`: Patch information is refreshed, but new patches are not applied to the kernel.
* `no-refresh`: Patch information is not refreshed.
* `stop`: The Livepatch daemon does not start.
