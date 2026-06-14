---
myst:
  html_meta:
    description: "Access Control - learn about this topic in Livepatch on-prem."
---


(server-explanation-access-control)=

# Access Control

Access to a Livepatch on-prem instance is gated such that clients are authenticated before they can download patches. Access control is managed by means of generated tokens. These tokens act as a way of both authenticating client machines and assigning a tier to each machine, i.e. determining how and when patches get rolled out.

See this [how-to](/server/how-to-guides/operations/use-livepatch-client-with-on-prem-server.md) to understand how to generate tokens.

For a technical reference of all authentication mechanisms (Basic Auth, Macaroons, resource tokens, sync tokens) and the cryptographic technologies they use, see [Authentication and authorization](/server/reference/authentication/authentication.md).
