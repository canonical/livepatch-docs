---
myst:
  html_meta:
    description: "How to enable client with Livepatch client."
---


(client-how-to-guides-how-to-enable-the-livepatch-client)=

# How to enable the Livepatch client

Livepatch is included in Ubuntu Pro (previously known as Ubuntu Advantage). The recommended way to install it is using the Ubuntu Pro client. The following commands enable live kernel patching and install the Livepatch client. See also [this tutorial](https://ubuntu.com/tutorials/enable-the-livepatch-service#1-overview).

```
# Attach your personal or enterprise subscription from ubuntu.com/pro
sudo pro attach

# Explicitly enable livepatch
sudo pro enable livepatch
```

This installs the Livepatch client and enrolls the system in the Ubuntu Livepatch service.
