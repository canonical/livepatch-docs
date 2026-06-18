---
myst:
  html_meta:
    description: "Reference listing the system information transmitted by the Livepatch Client to Canonical during periodic patch status checks."
---

(client-reference-data-sent-to-canonical)=

# Data sent to Canonical

The Livepatch Client sends periodic requests to servers hosted by Canonical to check for the availability of new patches. These requests are sent at a configurable interval (every 60 minutes by default) and include the following information:

- System architecture
- CPU model
- Kernel version
- Boot time and uptime
- Unique machine identifier, derived from `/etc/machine-id`
- Version of the currently applied live kernel patch, if any
- Current state of the system (whether a live kernel patch has been applied)
- Time of the last server request
- Livepatch Client version