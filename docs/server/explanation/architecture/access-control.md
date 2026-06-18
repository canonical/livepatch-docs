---
myst:
  html_meta:
    description: "Understand access control in the Livepatch on-premises server including token-based authentication, machine tiers, and rollout management."
---

(server-explanation-access-control)=

# Access control

> See also: {ref}`Authentication and authorization <server-reference-livepatch-server-authentication>`, {ref}`Use the Livepatch Client with the on-premises server <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>`

Access to a Livepatch on-premises instance is gated so that clients are authenticated before they can download patches. Access control is managed through generated tokens. These tokens authenticate client machines and assign a tier to each machine, determining how and when patches are rolled out.

See {ref}`Use the Livepatch Client with the on-premises server <server-how-to-guides-how-to-use-livepatch-client-with-an-on-prem-server>` to learn how to generate tokens.

For a technical reference of all authentication mechanisms -- Basic Auth, macaroons, resource tokens, and sync tokens -- and the cryptographic technologies they use, see {ref}`Authentication and authorization <server-reference-livepatch-server-authentication>`.