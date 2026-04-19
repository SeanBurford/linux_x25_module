# X.25 Module Synchronisation Object Catalogue

This document catalogues every synchronisation primitive in the x25 module, together with guidance on correct usage, current best-practice status, and observations relevant to later correctness analysis.

---

## 1. `x25_list_lock` — `rwlock_t`

**Definition:** `af_x25.c:68`
```c
DEFINE_RWLOCK(x25_list_lock);
```

### What it protects
The global socket hash list `x25_list` (declared at `af_x25.c:67` as `HLIST_HEAD(x25_list)`) and the per-socket `x25_sk(s)->neighbour` pointer (see §6 below for the unusual dual use).

### Current best-practice status
**Legacy — superseded by RCU.**  `rwlock_t` is a reader-writer spinlock: readers spin waiting for an active writer, and writers spin waiting for all readers to finish. Both sides busy-wait. Modern Linux networking code replaces this pattern with RCU for read-mostly linked lists:
- Readers use `rcu_read_lock()` / `rcu_read_unlock()` — truly lockless, no spinning, no cache-line ping-pong.
- Writers use `list_add_rcu()` / `list_del_rcu()` followed by `synchronize_rcu()` (or `call_rcu()` in BH context).

The socket list is read far more often than it is written (read on every inbound frame; written only on bind, connect, close, and interface-down events), making it a good RCU candidate.

### When to acquire
| Operation | Lock variant | Reason |
|---|---|---|
| Insert / remove a socket | `write_lock_bh` / `write_unlock_bh` | Exclusive list mutation |
| Look up a socket by address or LCI | `read_lock_bh` / `read_unlock_bh` | Concurrent readers OK |
| Iterate for `x25_kill_by_neigh` | `write_lock_bh` / `write_unlock_bh` | Avoids concurrent removal during kill walk |
| `/proc` iteration (`x25_seq_socket_start`) | `read_lock_bh` / `read_unlock_bh` | Read-only traversal |

The `_bh` suffix is mandatory: inbound frames arrive in softirq (bottom-half) context and also acquire this lock. Taking the lock without BH disabling in process context would allow a softirq to re-enter and deadlock.

### When not needed
- Accessing fields of a socket that is already held under `lock_sock` or `bh_lock_sock`. Once a reference to the socket has been obtained and the socket is locked, the socket is not going to be removed from the list in a way that affects those fields.
- Any code path that holds neither the list lock nor a socket reference must not dereference socket list entries.

### Multi-CPU performance guidance
- Keep the read critical section as short as possible: look up the socket, take a reference (`sock_hold`), release the read lock, then lock the socket.
- Never perform memory allocation, I/O, or sleeping operations while holding any variant of this lock — it is a spinlock.
- If converted to RCU: use `sk_for_each_rcu` inside `rcu_read_lock`, call `sock_hold` before `rcu_read_unlock`, then lock the socket.

### Call sites
| File | Line | Operation |
|---|---|---|
| `af_x25.c` | 197 | `write_lock_bh` — `x25_remove_socket` |
| `af_x25.c` | 199 | `write_unlock_bh` — `x25_remove_socket` |
| `af_x25.c` | 252 | `write_lock_bh` — `x25_insert_socket` |
| `af_x25.c` | 254 | `write_unlock_bh` — `x25_insert_socket` |
| `af_x25.c` | 270 | `read_lock_bh` — `x25_find_listener` |
| `af_x25.c` | 301 | `read_unlock_bh` — `x25_find_listener` |
| `af_x25.c` | 326 | `read_lock_bh` — `x25_find_socket` |
| `af_x25.c` | 328 | `read_unlock_bh` — `x25_find_socket` |
| `af_x25.c` | 832 | `read_lock_bh` — `x25_connect` |
| `af_x25.c` | 835 | `read_unlock_bh` — `x25_connect` |
| `af_x25.c` | 1775 | `write_lock_bh` — `x25_kill_by_neigh` |
| `af_x25.c` | 1779 | `write_unlock_bh` — dropped before `lock_sock` |
| `af_x25.c` | 1783 | `write_lock_bh` — re-acquired after `release_sock` |
| `af_x25.c` | 1786 | `write_unlock_bh` — `x25_kill_by_neigh` exit |
| `x25_subr.c` | 361 | `read_lock_bh` — protects `x25->neighbour` in `x25_disconnect` |
| `x25_subr.c` | 364 | `read_unlock_bh` — `x25_disconnect` |
| `x25_proc.c` | 63 | `read_lock_bh` — `/proc` socket sequence start |

### Notable usage pattern: `x25_kill_by_neigh` lock cycling
`x25_kill_by_neigh` (af_x25.c:1771) holds `write_lock_bh(&x25_list_lock)`, iterates the list, and for each matching socket: drops the lock, acquires `lock_sock`, calls `x25_disconnect`, releases `lock_sock`, then re-acquires the write lock to continue iteration. This is necessary to avoid lock-order inversion (`lock_sock` must not be taken inside the list spinlock). However, the list may have changed while the lock was dropped — sockets may have been added or removed at positions already past in the iteration. This is flagged as a potential race in the correctness analysis.

### Notable usage pattern: `x25_list_lock` protecting `x25->neighbour`
`x25_disconnect` (x25_subr.c:361) takes `read_lock_bh(&x25_list_lock)` to protect the write `x25->neighbour = NULL`. This is because `x25_kill_by_neigh` reads `x25_sk(s)->neighbour` while holding `write_lock_bh(&x25_list_lock)` but before acquiring `lock_sock`. The list lock is therefore used as a secondary guard for this specific per-socket field. This is an unusual dual-role for the lock.

---

## 2. `x25_neigh_list_lock` — `rwlock_t`

**Definition:** `x25_link.c:32`
```c
DEFINE_RWLOCK(x25_neigh_list_lock);
```

### What it protects
The global neighbour list `x25_neigh_list` (declared at `x25_link.c:30`) and per-neighbour mutable fields `nb->extended` and `nb->global_facil_mask`.

### Current best-practice status
**Legacy — superseded by RCU**, for the same reasons as `x25_list_lock`. The neighbour list is traversed on every inbound frame (via `x25_get_neigh`) and mutated only on interface up/down and ioctl.

### When to acquire
| Operation | Lock variant |
|---|---|
| Add or remove a neighbour entry | `write_lock_bh` / `write_unlock_bh` |
| Look up a neighbour by device | `read_lock_bh` / `read_unlock_bh` |
| Read `nb->extended` or `nb->global_facil_mask` | `read_lock_bh` / `read_unlock_bh` |
| Write `nb->extended` or `nb->global_facil_mask` | `write_lock_bh` / `write_unlock_bh` |
| Functions annotated "Caller must hold x25_neigh_list_lock" | caller holds; do not re-acquire |

`x25_link.c:297` carries the comment `/* Caller must hold x25_neigh_list_lock. */` — the internal helper `__x25_remove_neigh` must only be called with the write lock held.

### When not needed
Once a reference to an `x25_neigh` has been obtained via `x25_get_neigh` (which calls `x25_neigh_hold` inside the read lock), the caller owns a reference and may safely dereference the neighbour pointer without holding the list lock, provided it is not modifying the list itself.

### Multi-CPU performance guidance
Same as `x25_list_lock`: take the lock only long enough to locate the entry and increment its refcount, then release before doing heavier work.

### Call sites
| File | Line | Operation |
|---|---|---|
| `x25_link.c` | 287 | `write_lock_bh` — `x25_link_device_up` |
| `x25_link.c` | 289 | `write_unlock_bh` — `x25_link_device_up` |
| `x25_link.c` | 315 | `write_lock_bh` — `x25_link_device_down` |
| `x25_link.c` | 326 | `write_unlock_bh` — `x25_link_device_down` |
| `x25_link.c` | 336 | `read_lock_bh` — `x25_get_neigh` |
| `x25_link.c` | 346 | `read_unlock_bh` — `x25_get_neigh` |
| `x25_link.c` | 377 | `read_lock_bh` — `x25_subscr_ioctl` (GET path) |
| `x25_link.c` | 380 | `read_unlock_bh` — `x25_subscr_ioctl` (GET path) |
| `x25_link.c` | 387 | `write_lock_bh` — `x25_subscr_ioctl` (SET path) |
| `x25_link.c` | 390 | `write_unlock_bh` — `x25_subscr_ioctl` (SET path) |
| `x25_link.c` | 410 | `write_lock_bh` — `x25_link_free` |
| `x25_link.c` | 420 | `write_unlock_bh` — `x25_link_free` |
| `af_x25.c` | 1654 | `read_lock_bh` — `compat_x25_subscr_ioctl` (GET path) |
| `af_x25.c` | 1657 | `read_unlock_bh` — `compat_x25_subscr_ioctl` (GET path) |
| `af_x25.c` | 1664 | `write_lock_bh` — `compat_x25_subscr_ioctl` (SET path) |
| `af_x25.c` | 1667 | `write_unlock_bh` — `compat_x25_subscr_ioctl` (SET path) |

---

## 3. `x25_route_list_lock` — `rwlock_t`

**Definition:** `x25_route.c:21`
```c
DEFINE_RWLOCK(x25_route_list_lock);
```

### What it protects
The global route list `x25_route_list` (declared at `x25_route.c:19`) and the address/device mappings within each `x25_route` entry.

`x25_route.c:64` carries the annotation `/* Caller must hold x25_route_list_lock. */` for the internal `__x25_remove_route` helper.

### Current best-practice status
**Legacy — superseded by RCU.** Routes are read on every outbound connection setup and mutated only by ioctl.

### When to acquire
| Operation | Lock variant |
|---|---|
| Add or remove a route | `write_lock_bh` / `write_unlock_bh` |
| Look up a route by address | `read_lock_bh` / `read_unlock_bh` |
| `/proc` iteration | `read_lock_bh` / `read_unlock_bh` |

### When not needed
After obtaining a route reference via `x25_get_route` (which holds inside the read lock), the caller may use the route without holding the list lock.

### Call sites
| File | Line | Operation |
|---|---|---|
| `x25_route.c` | 32 | `write_lock_bh` — `x25_add_route` |
| `x25_route.c` | 55 | `write_unlock_bh` — `x25_add_route` |
| `x25_route.c` | 80 | `write_lock_bh` — `x25_del_route` |
| `x25_route.c` | 91 | `write_unlock_bh` — `x25_del_route` |
| `x25_route.c` | 103 | `write_lock_bh` — `x25_route_device_down` |
| `x25_route.c` | 111 | `write_unlock_bh` — `x25_route_device_down` |
| `x25_route.c` | 139 | `read_lock_bh` — `x25_get_route` |
| `x25_route.c` | 153 | `read_unlock_bh` — `x25_get_route` |
| `x25_route.c` | 198 | `write_lock_bh` — `x25_route_free` |
| `x25_route.c` | 203 | `write_unlock_bh` — `x25_route_free` |
| `x25_proc.c` | 28 | `read_lock_bh` — `/proc` route sequence start |

---

## 4. `x25_forward_list_lock` — `rwlock_t`

**Definition:** `x25_forward.c:15`
```c
DEFINE_RWLOCK(x25_forward_list_lock);
```

### What it protects
The global forward list `x25_forward_list` (declared at `x25_forward.c:13`) and the LCI / device pair mappings within each `x25_forward` entry.

### Current best-practice status
**Legacy — superseded by RCU**, same rationale as the other three list locks. Forwarding lookups happen on the inbound data path (per packet), mutations only on call setup and teardown.

### When to acquire
| Operation | Lock variant |
|---|---|
| Check if a call already exists before adding | `read_lock_bh` / `read_unlock_bh` |
| Add a new forward entry | `write_lock_bh` / `write_unlock_bh` |
| Remove entries by LCI or device | `write_lock_bh` / `write_unlock_bh` |
| Forward a data frame (lookup only) | `read_lock_bh` / `read_unlock_bh` |
| `/proc` iteration | `read_lock_bh` / `read_unlock_bh` |

### When not needed
The `x25_forward.c:47–54` pattern — read-lock to check for duplicates, then release, then write-lock to insert — is a classic check-then-act race window. The duplicate check and insertion should ideally be done under a single write lock.

### Call sites
| File | Line | Operation |
|---|---|---|
| `x25_forward.c` | 47 | `read_lock_bh` — duplicate check in `x25_forward_call` |
| `x25_forward.c` | 54 | `read_unlock_bh` — duplicate check done |
| `x25_forward.c` | 66 | `write_lock_bh` — insert in `x25_forward_call` |
| `x25_forward.c` | 68 | `write_unlock_bh` — insert done |
| `x25_forward.c` | 98 | `read_lock_bh` — lookup in `x25_forward_data` |
| `x25_forward.c` | 110 | `read_unlock_bh` — lookup done |
| `x25_forward.c` | 132 | `write_lock_bh` — `x25_clear_forward_by_lci` |
| `x25_forward.c` | 140 | `write_unlock_bh` — `x25_clear_forward_by_lci` |
| `x25_forward.c` | 148 | `write_lock_bh` — `x25_clear_forward_by_dev` |
| `x25_forward.c` | 156 | `write_unlock_bh` — `x25_clear_forward_by_dev` |
| `x25_proc.c` | 115 | `read_lock_bh` — `/proc` forward sequence start |

---

## 5. `lock_sock` / `release_sock` — per-socket process-context lock

**Mechanism:** Kernel socket lock (`sk->sk_lock`). On modern kernels this is a mutex-like construct that also tracks whether the socket is owned by a user-context thread, allowing BH-context code to queue backlog frames rather than dead-locking.

**Not declared in this codebase** — provided by the kernel's socket infrastructure (`include/net/sock.h`).

### What it protects
All mutable state of a specific `struct sock` / `struct x25_sock` when accessed from process (user) context:

- `sk->sk_state` and the X.25-level state `x25->state`
- Socket flags (`SOCK_DEAD`, `SOCK_DESTROY`, `SOCK_ZAPPED`, etc.)
- X.25 sequence variables: `x25->vs`, `x25->vr`, `x25->va`, `x25->vl`
- `x25->condition`, `x25->facilities`, `x25->calluserdata`, `x25->causediag`
- Per-socket queue: `sk->sk_receive_queue`, `x25->interrupt_in_queue`, `x25->fragment_queue`, `x25->ack_queue`
- `x25->neighbour` (together with `x25_list_lock` for the specific field described in §1)
- Binding state: `x25->source_addr`, `x25->dest_addr`, `x25->lci`

### Current best-practice status
**Current and correct.** `lock_sock` / `release_sock` remain the standard kernel API for per-socket locking in process context. No deprecation.

### When to invoke
Acquire before any read or write of the socket fields listed above from system call, ioctl, or other process-context paths. The lock must be held for the duration of any state machine transition. All `x25_ioctl` cases in `af_x25.c` (lines 1403–1611) follow this pattern.

The pattern in `x25_sendmsg` (af_x25.c:1115, 1178–1180) of releasing and re-acquiring around `sock_alloc_send_skb` is correct: `sock_alloc_send_skb` may sleep, so the lock must be dropped first.

### When not needed
- From timer expiry handlers or softirq context — use `bh_lock_sock` instead (§6).
- When only accessing fields protected by a different lock (e.g., traversing the global socket list under `x25_list_lock`).
- After the socket has been fully closed and removed from all lists.

### Multi-CPU performance guidance
- Drop the lock before any potentially sleeping call (`sock_alloc_send_skb`, `copy_to_user`, etc.) and re-acquire afterwards.
- Minimise the locked window. State transitions should be brief.

### Call sites (selected)
| File | Lines | Context |
|---|---|---|
| `af_x25.c` | 484, 487, 497 | `x25_bind` — check/set bind state |
| `af_x25.c` | 636–668 | `x25_connect` — connection state machine |
| `af_x25.c` | 698–706 | `x25_accept` — wait for incoming connection |
| `af_x25.c` | 735, 737 | `x25_accept` — wait loop re-acquire |
| `af_x25.c` | 755–841 | `x25_getname` |
| `af_x25.c` | 1115–1269 | `x25_sendmsg` — with mid-function release for alloc |
| `af_x25.c` | 1289–1373 | `x25_recvmsg` |
| `af_x25.c` | 1403–1611 | All `x25_ioctl` cases |
| `af_x25.c` | 1780–1782 | `x25_kill_by_neigh` — after dropping list lock |
| `x25_out.c` | 66, 69 | Dropped/re-acquired around `sock_alloc_send_skb` |

---

## 6. `bh_lock_sock` / `bh_unlock_sock` — per-socket BH-context lock

**Mechanism:** The fast-path half of the socket lock. Acquires `sk->sk_lock.slock` (a raw spinlock) without sleeping. If the socket is already owned by a process-context thread (`sock_owned_by_user` returns true), the caller must queue the work to backlog rather than processing it inline.

**Not declared in this codebase** — kernel infrastructure.

### What it protects
Same socket state as `lock_sock`, but accessed from softirq (bottom-half) or timer context where sleeping is forbidden.

### Current best-practice status
**Current and correct.** The `bh_lock_sock` / `sock_owned_by_user` / backlog pattern is the established kernel idiom for handling packets arriving in softirq context.

### When to invoke
- In timer expiry callbacks (`x25_heartbeat_expiry`, `x25_timer_expiry` in `x25_timer.c`).
- In the inbound data path (`x25_receive_data` in `x25_dev.c`) when a matching socket has been found.
- In `x25_destroy_socket_from_timer` (af_x25.c:412) which is called from a timer.

The mandatory pattern is:
```c
bh_lock_sock(sk);
if (sock_owned_by_user(sk)) {
    /* queue to backlog — process context will drain it */
} else {
    /* process frame directly */
}
bh_unlock_sock(sk);
```

### When not needed
From process context — use `lock_sock` / `release_sock` instead. Using `bh_lock_sock` from process context would not disable BH and could race with a softirq taking the same lock.

### Multi-CPU performance guidance
- Keep the critical section minimal; timer handlers should avoid any heavy work.
- Always check `sock_owned_by_user` and queue to backlog if true, never spin waiting for the process-context lock.

### Call sites
| File | Lines | Context |
|---|---|---|
| `af_x25.c` | 412–414 | `x25_destroy_socket_from_timer` |
| `x25_dev.c` | 54, 60 | `x25_receive_data` — inbound frame |
| `x25_timer.c` | 94, 124 | `x25_heartbeat_expiry` |
| `x25_timer.c` | 162, 168 | `x25_timer_expiry` |

---

## 7. `refcount_t` — reference counts on shared objects

**Instances:**
- `nb->refcnt` in `struct x25_neigh` — initialised at `x25_link.c:285`
- `rt->refcnt` in `struct x25_route` — initialised at `x25_route.c:50`

**Not a synchronisation object in the lock sense**, but essential for lifetime management: an object is only freed when its refcount reaches zero. `refcount_t` (as opposed to the older `atomic_t`) triggers a kernel warning on overflow or underflow.

**Best practice:** `refcount_t` is the current correct API (supersedes raw `atomic_t` for reference counts). Correct usage requires that any code holding a pointer to an `x25_neigh` or `x25_route` has incremented the refcount (via `x25_neigh_hold` / `x25_route_hold`) and decrements it when done (via `x25_neigh_put` / `x25_route_put`).

---

## 8. `atomic_t` reads — receive-buffer occupancy

**Instances:**
- `x25_in.c:291` — `atomic_read(&sk->sk_rmem_alloc)` to decide whether to set `X25_COND_OWN_RX_BUSY`
- `x25_subr.c:376` — `atomic_read(&sk->sk_rmem_alloc)` in `x25_check_rbuf`

These are read-only accesses of a kernel-maintained atomic counter; no locking is required around them beyond what the socket lock already provides for the subsequent condition flag updates.

---

## Lock ordering

When multiple locks must be held simultaneously the following order must be respected to prevent deadlock:

```
x25_list_lock  (global, outermost)
  └─ lock_sock / bh_lock_sock  (per socket, innermost)
```

`x25_kill_by_neigh` demonstrates the required discipline: the global list write lock is dropped **before** `lock_sock` is acquired, and re-acquired **after** `release_sock`. The three other global list locks (`x25_neigh_list_lock`, `x25_route_list_lock`, `x25_forward_list_lock`) are never held simultaneously with each other or with `x25_list_lock` in the current code.

---

## Summary table

| Object | Type | Defined in | Superseded? | Protects |
|---|---|---|---|---|
| `x25_list_lock` | `rwlock_t` | `af_x25.c:68` | Yes — prefer RCU | `x25_list` hlist; `x25->neighbour` field |
| `x25_neigh_list_lock` | `rwlock_t` | `x25_link.c:32` | Yes — prefer RCU | `x25_neigh_list`; `nb->extended`, `nb->global_facil_mask` |
| `x25_route_list_lock` | `rwlock_t` | `x25_route.c:21` | Yes — prefer RCU | `x25_route_list` |
| `x25_forward_list_lock` | `rwlock_t` | `x25_forward.c:15` | Yes — prefer RCU | `x25_forward_list` |
| `lock_sock` / `release_sock` | socket lock | kernel infra | No | All per-socket state (process context) |
| `bh_lock_sock` / `bh_unlock_sock` | socket BH lock | kernel infra | No | All per-socket state (BH/timer context) |
| `nb->refcnt`, `rt->refcnt` | `refcount_t` | per-object | No (current API) | Object lifetime |
| `sk->sk_rmem_alloc` reads | `atomic_t` | kernel infra | N/A | Read-only metric |
