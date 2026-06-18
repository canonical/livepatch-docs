---
myst:
  html_meta:
    description: "Understand Livepatch tiers for patch delivery, including Proposed, Internal, Updates, and Stable tiers and how they relate to Ubuntu Pro subscriptions."
---

(client-explanation-what-are-livepatch-tiers)=

# Livepatch tiers

Livepatch delivers patches through a tiered system. A tier is a target audience for patch delivery. Your tier depends on whether you have a free or paid [Ubuntu Pro](https://ubuntu.com/pro) subscription.

| Tier | Description |
| :--- | :--- |
| Proposed | The initial tier where patches are added before promotion to subsequent tiers. |
| Internal | For internal Canonical use. Patches are tested and applied across Canonical infrastructure to reduce the risk of a faulty patch reaching customer machines. |
| Updates | For free users of Ubuntu Pro. Patches are delivered to these machines after the Internal tier. |
| Stable | For paid users of Ubuntu Pro. Patches are delivered to these machines last. |

The kernel team monitors patch health within the Internal tier before promoting to the Updates tier, and performs further monitoring before promoting to the Stable tier.

For finer-grained control over patch rollout, see the [Livepatch On-Prem documentation](/server/index.md).
