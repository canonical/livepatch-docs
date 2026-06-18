---
myst:
  html_meta:
    description: "Understand how Canonical rates CVEs in Livepatch, including the rating scale from negligible to critical and what each rating means."
---

(client-explanation-how-do-you-rate-a-cve)=

# How CVEs are rated

Canonical uses its own rating system for CVEs based on the following qualifications:

| Rating | Description |
|--------|-------------|
| Negligible | A theoretical security problem that requires a very specific scenario, has a minimal install base, or causes negligible damage. These usually do not receive backports from upstream and are unlikely to be included in security updates unless an easy fix is available and another issue triggers an update. |
| Low | A security problem that is difficult to exploit due to environment constraints, requires a user-assisted attack, has a small install base, or causes limited damage. These are typically included in security updates only when higher priority issues require an update or when many low priority issues have accumulated. |
| Medium | A real security problem that is exploitable for many users. Includes network daemon denial of service attacks, cross-site scripting, and gaining user privileges. Updates should be produced promptly for this priority. |
| High | A real problem, exploitable for many users in a default installation. Includes serious remote denial of service, local root privilege escalation, or data loss. |
| Critical | A severe problem, exploitable for nearly all users in a default installation of Ubuntu. Includes remote root privilege escalation or massive data loss. |
