---
myst:
  html_meta:
    description: "How to generate patch health report with Livepatch on-prem."
---

(server-how-to-guides-how-to-generate-livepatch-on-prem-patch-health-report)=

# Generate a Livepatch on-premises patch health report

To generate a report of how many machines applied (or failed to apply) a specific patch, run:

```
livepatch-admin report patch-health <tier> <patch-version>
```