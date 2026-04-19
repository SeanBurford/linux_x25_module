# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an out-of-tree Linux kernel module for the X.25 Packet Layer Protocol, sourced from `/net/x25/` in the Linux 6.12 kernel tree (Debian package). The only local addition is `dkms.conf` for simplified out-of-tree builds.

## Build

```bash
cd x25-6.12.74+deb13+1-amd64/
make
```

Produces `x25.ko`. The module compiles against the running kernel's headers via the standard `$(MAKE) -C /lib/modules/$(shell uname -r)/build` mechanism.

The Makefile uses `-include $(src)/include/net/x25.h` to force-include a local copy of `x25.h` that overrides the system kernel header. This is necessary because the module uses `spinlock_t` for its four global list locks, while the installed kernel headers declare them as `rwlock_t`. The local header at `include/net/x25.h` must be kept in sync with `net/x25.h` from the kernel headers, with only the four lock extern declarations changed to `spinlock_t`.

**DKMS workflow:**
```bash
dkms add -m x25 -v 6.12.27-amd64
dkms build -m x25 -v 6.12.27-amd64
dkms install -m x25 -v 6.12.27-amd64
```

## Load / Unload

```bash
sudo insmod x25-6.12.74+deb13+1-amd64/x25.ko
sudo rmmod x25
```

## Runtime Inspection

```bash
cat /proc/net/x25/route    # routing table
cat /proc/net/x25/socket   # active sockets
sysctl -a | grep x25       # tunable timeouts
```

## Architecture

All source lives under `x25-6.12.74+deb13+1-amd64/`. The module registers `AF_X25`/`PF_X25` socket family and a packet type handler with the kernel.

**Inbound data path:**
```
Network device → x25_dev.c (x25_lapb_receive_frame)
              → af_x25.c (x25_find_socket / x25_rx_call_request)
              → x25_in.c (state machine dispatch)
              → socket receive queue → application recvmsg
```

**Key files:**
| File | Role |
|------|------|
| `af_x25.c` | Socket family entry point: bind/connect/listen/accept/send/recv, module init/exit |
| `x25_in.c` | Inbound frame state machine (call setup, data transfer, teardown) |
| `x25_out.c` | Outbound frame transmission and packet fragmentation |
| `x25_link.c` | Neighbor/link management, restart (T20) timer, link state 0-3 |
| `x25_dev.c` | Kernel packet-type registration; bridges LAPB device frames into X.25 |
| `x25_route.c` | Routing table: maps X.25 addresses to net devices |
| `x25_facilities.c` | Facility option parsing and negotiation (window/packet size, throughput) |
| `x25_forward.c` | Call forwarding table between interfaces |
| `x25_timer.c` | T21 (call request), T22 (reset), T23 (clear), T2 (ack holdback) timers |
| `x25_subr.c` | Shared queue helpers and frame utilities |
| `x25_proc.c` | `/proc/net/x25/` entries |
| `sysctl_net_x25.c` | `/proc/sys/net/x25/` sysctl knobs |

**Core data structures** (defined in kernel headers, used throughout):
- `x25_sock` — per-socket protocol state and LCI
- `x25_neigh` — neighbor/link entry
- `x25_route` — routing table entry
- `x25_facilities` — negotiated call facilities
- `x25_forward` — forwarding table entry

## Sysctl Tunables

Configured via `/proc/sys/net/x25/`:
- `call_request_timeout` (T21)
- `reset_request_timeout` (T22)
- `clear_request_timeout` (T23)
- `acknowledgement_hold_back_timeout` (T2)
- `forward` — enable call forwarding
