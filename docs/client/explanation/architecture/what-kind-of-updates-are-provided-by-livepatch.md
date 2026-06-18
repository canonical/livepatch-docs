---
myst:
  html_meta:
    description: "Understand which updates the Livepatch service provides, including high and critical CVE fixes for Linux kernel vulnerabilities delivered as cumulative live kernel patches."
---

(client-explanation-what-kind-of-updates-will-be-provided-by-the-ubuntu-livepatch-service)=

# Updates provided by Livepatch

The Livepatch service addresses high and critical severity Linux kernel security vulnerabilities, as identified by [Ubuntu Security Notices](https://ubuntu.com/security/notices) and the [CVE tracker](https://ubuntu.com/security/cve). Due to limitations of the [kernel livepatch technology](https://github.com/torvalds/linux/blob/master/Documentation/livepatch/livepatch.rst), some Linux kernel code paths cannot be safely patched while running. In these cases, a traditional kernel upgrade and reboot is necessary.

Live kernel patches are developed and released based on kernel SRU releases, and contain a subset of the CVEs fixed by each kernel SRU. Patches are cumulative — when a new live kernel patch is released, it contains both the new CVE fixes and all CVE fixes from the previous release of that patch.
