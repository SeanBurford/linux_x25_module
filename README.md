# Linux x.25 module

Copied from `/net/x25/` after fetching the Linux kernel source using Debian `apt source linux`

## Local Changes

*  Addition of `dkms.conf` to simplify out of tree builds.
*  Addition of synchronisation analysis under /analysis.
*  Fix SYNC-001 — Write operation performed under read lock on `x25->neighbour`.
*  Fix SYNC-002 — TOCTOU in `x25_forward_call`: duplicate-check and insert are not atomic.
*  Fix SYNC-003 — `x25_kill_by_neigh` drops the list write lock mid-iteration, risking stale next-pointer.
