# Linux x.25 module

Copied from `/net/x25/` after fetching the Linux kernel source using Debian `apt source linux`

## Local Changes

*  Addition of `dkms.conf` to simplify out of tree builds.
*  Addition of synchronisation analysis under `/analysis`.
*  Fix for SYNC-001 — Write operation performed under read lock on x25-\>neighbour.
*  Fix for SYNC-002 — TOCTOU in x25\_forward\_call: duplicate-check and insert are not atomic
