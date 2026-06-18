---
myst:
  html_meta:
    description: "Reference for Livepatch on-premises server resource requirements, listing minimum CPU, memory, and disk allocations for PostgreSQL, HAProxy, and Livepatch components."
---

(server-reference-machine-resources-for-livepatch-on-prem)=

# Machine resources for Livepatch on-prem

The resource requirements for machines running Livepatch on-prem components (PostgreSQL, HAProxy, and Livepatch) depend on the number of machines served by the deployment.

The following are the minimum recommended allocations:

| Service    | Memory | CPUs | Disk |
| ---------- | ------ | ---- | ---- |
| PostgreSQL | 4 GB   | 1    | 50 GB |
| HAProxy    | 2 GB   | 1    | 50 GB |
| Livepatch  | 2 GB   | 1    | 50 GB |

If PostgreSQL is configured as the patch storage backend, increase the disk allocation for PostgreSQL to 100 GB.

The bundle must be deployed on machines running Ubuntu Focal (20.04 LTS).