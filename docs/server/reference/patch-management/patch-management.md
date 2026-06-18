---
myst:
  html_meta:
    description: "Technical reference for managing patches in an on-premises Livepatch deployment, covering tier-based patch rollout and promotion using livepatch-admin."
---

(server-reference-managing-patches-in-an-on-prem-livepatch-deployment)=

# Managing patches in an on-prem Livepatch deployment

Livepatch patches are managed using tiers — ordered groups that patches are placed into. Each Livepatch Client instance is associated with a specific tier through its token. After deployment, the server includes a default tier list: `edge`, `beta`, `candidate`, and `stable`. This list can be modified using the Livepatch administration tool.

![Patch tier overview](/_static/images/a5BXTUhxJVYx45D4Bb4Av7VtVW3.png)

The tiers form an ordered list. When the on-premises server pulls patches from Canonical's servers, patches are initially assigned to the first tier in the list. The order of tiers in the output of the `livepatch-admin` tool reflects the order of patch promotion.

To view the current tier order:

```
livepatch-admin tier list
```

The output resembles:

```
Tiers:
- Name: edge
- Name: updates
- Name: stable
- Name: <on-prem>
```

If no further validation of patches is required, associate all Livepatch Client instances with the first tier. Patches become available as soon as they have been downloaded.

![Single-tier deployment](/_static/images/z7xA042h7EvRf7fRFNqLGiGi9Mw.png)

If validation or a staggered rollout of patches is required, use the patch tiers to implement this workflow. Associate a small portion of testing machines with the `beta` tier, and promote patches to higher tiers once they have been validated.

![Multi-tier promotion](/_static/images/hJiubhhVct8K28ljnzbxYhdyjza.png)

## Promote a patch to a different tier

To promote an individual patch to a different tier:

```
livepatch-admin patch promote <patch-version> <tier>
```

The `patch-version` is the numerical patch version (for example, `57.1`).

## Promote all patches in a tier

To promote all patches from one tier to another, use the `promote-all` shortcut:

```
livepatch-admin patch promote-all <from-tier> <to-tier>
```

This command is useful during initial setup of Livepatch on-prem, when all patches are imported into the `edge` tier and other tiers are empty. For example:

```text
livepatch-admin patch promote-all edge stable
```

This makes the contents of all tiers from `edge` to `stable` identical.