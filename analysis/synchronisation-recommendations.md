# X.25 Module Synchronisation Recommendations

Recommendations are ordered by priority: higher impact and simpler fixes first. Each entry cites the exact files and lines from which the problem was identified; the catalogue in `synchronisation.md` provides the broader lock context.

---

## Priority 1 — Critical: Data Races and TOCTOU with Simple Fixes

### SYNC-001 — Write operation performed under read lock on `x25->neighbour`

**Summary:** `x25_disconnect` and the `x25_connect` error path write `x25->neighbour = NULL` while holding only `read_lock_bh(&x25_list_lock)`, allowing concurrent reads (including from `/proc`) to race with the write.

**Problem:** A reader-writer lock guarantees mutual exclusion only when writers hold the write lock. Holding a read lock while performing a write gives no exclusion against other readers. Two CPUs can therefore simultaneously:
- CPU A: execute `x25_seq_socket_show` reading `x25->neighbour` under `read_lock_bh`
- CPU B: execute `x25_disconnect` writing `x25->neighbour = NULL` under `read_lock_bh`

The `/proc` path then potentially dereferences a pointer being zeroed — a use-after-free or null-pointer dereference.

**Affected code:**
- `x25_subr.c:361–364` — `x25_disconnect`, writes `x25->neighbour = NULL`
- `af_x25.c:832–835` — `x25_connect` error path, writes `x25->neighbour = NULL`

**Proposed change:** Replace `read_lock_bh` / `read_unlock_bh` with `write_lock_bh` / `write_unlock_bh` at both sites. This correctly serialises the write against concurrent readers, including `/proc` iteration.

```c
/* x25_subr.c and af_x25.c — change at both sites: */
- read_lock_bh(&x25_list_lock);
  x25_neigh_put(x25->neighbour);
  x25->neighbour = NULL;
- read_unlock_bh(&x25_list_lock);
+ write_lock_bh(&x25_list_lock);
  x25_neigh_put(x25->neighbour);
  x25->neighbour = NULL;
+ write_unlock_bh(&x25_list_lock);
```

Note: both call sites already hold `lock_sock`. The write lock is taken inside the socket lock. This ordering (`lock_sock` → `write_lock_bh(x25_list_lock)`) is already present in the codebase — `x25_new_lci` → `x25_find_socket` does `lock_sock` → `read_lock_bh`. No new lock-ordering relationship is introduced; the change only strengthens the lock variant at the write sites.

**Estimated lines changed:** 4 (2 per site × 2 sites)

**Affected functions:** `x25_disconnect` (`x25_subr.c`), `x25_connect` (`af_x25.c`)

---

### SYNC-002 — TOCTOU in `x25_forward_call`: duplicate-check and insert are not atomic

**Summary:** `x25_forward_call` checks for a duplicate LCI under a read lock, releases it, then inserts under a write lock. Two concurrent callers can both pass the check and both insert an entry for the same LCI.

**Problem:** The current pattern in `x25_forward.c:47–68`:

```c
read_lock_bh(&x25_forward_list_lock);
list_for_each_entry(...)  /* check for duplicate */
    same_lci = 1;
read_unlock_bh(&x25_forward_list_lock);
/* ← WINDOW: another thread can also pass the check here */
if (!same_lci) {
    new_frwd = kmalloc(...);
    write_lock_bh(&x25_forward_list_lock);
    list_add(&new_frwd->node, &x25_forward_list);
    write_unlock_bh(&x25_forward_list_lock);
}
```

Two concurrent call requests for the same LCI arriving on different CPUs can both observe `same_lci == 0` and both insert an entry. Subsequent forwarding lookups in `x25_forward_data` will then match unpredictably between the two entries.

**Proposed change:** Move the duplicate check inside the write lock so that the check and insert form a single atomic operation. The `kmalloc(GFP_ATOMIC)` can be done before acquiring the lock to avoid allocating inside the critical section.

```c
new_frwd = kmalloc(sizeof(*new_frwd), GFP_ATOMIC);
if (!new_frwd) { rc = -ENOMEM; goto out_put_nb; }
new_frwd->lci  = lci;
new_frwd->dev1 = rt->dev;
new_frwd->dev2 = from->dev;

write_lock_bh(&x25_forward_list_lock);
list_for_each_entry(x25_frwd, &x25_forward_list, node) {
    if (x25_frwd->lci == lci) {
        pr_warn("...");
        same_lci = 1;
        break;
    }
}
if (!same_lci)
    list_add(&new_frwd->node, &x25_forward_list);
write_unlock_bh(&x25_forward_list_lock);

if (same_lci)
    kfree(new_frwd);
```

**Estimated lines changed:** ~12

**Affected functions:** `x25_forward_call` (`x25_forward.c`)

---

## Priority 2 — High: Race Conditions Requiring Moderate Restructuring

### SYNC-003 — `x25_kill_by_neigh` drops the list write lock mid-iteration, risking stale next-pointer

**Summary:** `x25_kill_by_neigh` iterates `x25_list` under `write_lock_bh`, drops the lock to call `lock_sock`, then re-acquires the lock and continues from the same list position. If the list changes during the window the next-pointer may have been modified or a successor removed.

**Problem:** `af_x25.c:1775–1786`:

```c
write_lock_bh(&x25_list_lock);
sk_for_each(s, &x25_list) {
    if (x25_sk(s)->neighbour == nb) {
        write_unlock_bh(&x25_list_lock);   /* list can change here */
        lock_sock(s);
        x25_disconnect(s, ENETUNREACH, 0, 0);
        release_sock(s);
        write_lock_bh(&x25_list_lock);     /* resumes from s->sk_node.next */
    }
}
```

`x25_disconnect` does not remove `s` from `x25_list`, so `s->sk_node.next` should still be valid for the current socket. However:
- A socket earlier in the hash list that was inserted during the window will be missed (not disconnected).
- A socket at the current position that is concurrently destroyed (e.g., by `x25_destroy_socket_from_timer`) could invalidate `s->sk_node.next`.

The safest fix is the kernel-idiomatic restart-after-process pattern.

**Proposed change:** Take a reference on `s` before dropping the lock; after releasing the socket lock, go back to the start of the list. The neighbour teardown path is infrequent so O(n²) is acceptable.

```c
restart:
    write_lock_bh(&x25_list_lock);
    sk_for_each(s, &x25_list) {
        if (x25_sk(s)->neighbour == nb) {
            sock_hold(s);
            write_unlock_bh(&x25_list_lock);
            lock_sock(s);
            x25_disconnect(s, ENETUNREACH, 0, 0);
            release_sock(s);
            sock_put(s);
            goto restart;
        }
    }
    write_unlock_bh(&x25_list_lock);
```

**Estimated lines changed:** ~8

**Affected functions:** `x25_kill_by_neigh` (`af_x25.c`)

---

### SYNC-004 — TOCTOU in `x25_new_lci`: LCI found-free then assigned without holding an exclusive lock

**Summary:** `x25_new_lci` iterates the socket list under a read lock to find a free LCI, returns it, and the caller assigns it — but the read lock is released between finding the free slot and recording it in `x25->lci`. Two concurrent `x25_connect` calls to the same neighbour can get the same LCI.

**Problem:** `af_x25.c:335–350` and caller at `af_x25.c:795`:

```c
/* Inside x25_connect, holding lock_sock(sk): */
x25->lci = x25_new_lci(x25->neighbour);
```

`x25_new_lci` calls `x25_find_socket(lci, nb)` in a loop; each call takes and releases `read_lock_bh`. After `x25_find_socket` returns NULL (LCI is free) and the loop exits, no lock is held. A second thread doing the same search will also see the LCI as free because `x25->lci` for `sk` is still 0 (the socket is in `x25_list` but not yet assigned the chosen LCI). Both threads then assign the same LCI.

**Proposed change:** Rewrite `x25_new_lci` to perform the entire scan under `write_lock_bh(&x25_list_lock)` and assign `x25->lci` inside the lock before releasing it. This requires passing `sk` itself (to fill in the LCI atomically) rather than returning the value. To avoid calling `x25_find_socket` (which takes its own lock), scan the list directly:

```c
static unsigned int x25_new_lci(struct sock *owner, struct x25_neigh *nb)
{
    unsigned int lci = 1;
    struct sock *s;

    write_lock_bh(&x25_list_lock);
    while (lci < 4096) {
        bool in_use = false;
        sk_for_each(s, &x25_list) {
            if (s != owner &&
                x25_sk(s)->lci == lci &&
                x25_sk(s)->neighbour == nb) {
                in_use = true;
                break;
            }
        }
        if (!in_use) {
            x25_sk(owner)->lci = lci;
            break;
        }
        lci++;
    }
    write_unlock_bh(&x25_list_lock);
    return (lci < 4096) ? lci : 0;
}
```

The caller in `x25_connect` does not separately assign `x25->lci`; the function does so atomically under the write lock. Remove the `cond_resched()` since the loop now runs under a spinlock.

**Estimated lines changed:** ~25 (rewrite of `x25_new_lci` + update of call site)

**Affected functions:** `x25_new_lci` (`af_x25.c`), `x25_connect` (`af_x25.c`)

---

### SYNC-005 — `x25_getname` reads socket address fields without any lock

**Summary:** `x25_getname` reads `sk->sk_state` and copies `x25->dest_addr` / `x25->source_addr` without holding `lock_sock`. These fields are written from `x25_connect` and `x25_disconnect` under `lock_sock`.

**Problem:** `af_x25.c:916–938` performs:

```c
if (sk->sk_state != TCP_ESTABLISHED)   /* sk_state: no lock */
    return -ENOTCONN;
sx25->sx25_addr = x25->dest_addr;      /* 16-byte copy: no lock */
```

A concurrent `x25_disconnect` on another CPU can set `sk_state = TCP_CLOSE` and zero `dest_addr`/`source_addr` while `x25_getname` is in the middle of copying. The result is a torn read of the 16-byte address. This is a benign-looking but real race.

**Proposed change:** Wrap the function body with `lock_sock` / `release_sock`, consistent with all other socket syscall handlers in this file. The critical section is small and does no sleeping work.

**Estimated lines changed:** 4 (add `lock_sock` at entry, `release_sock` before each return)

**Affected functions:** `x25_getname` (`af_x25.c`)

---

## Priority 3 — Medium: Design Improvement

### SYNC-006 — Eliminate the dual role of `x25_list_lock` as a guard for `x25->neighbour`

**Summary:** `x25_list_lock` serves two unrelated purposes: (1) protecting the socket hash list, and (2) acting as a secondary guard for the per-socket `x25->neighbour` field. This dual role is not documented in the code and creates a subtle invariant that future maintainers may violate.

**Background:** The dual role exists because `x25_kill_by_neigh` reads `x25_sk(s)->neighbour` while holding the list write lock (before acquiring `lock_sock`). To make that read safe, `x25_disconnect` and `x25_connect` also hold a list lock when writing `x25->neighbour`. After SYNC-001, the write sites correctly use `write_lock_bh`, so the mechanism is technically sound — but it remains confusing.

**Proposed change:** Change `x25_kill_by_neigh` to avoid reading `x25->neighbour` before taking `lock_sock`. Instead, check `neighbour` inside `lock_sock` and disconnect only if the neighbour matches. This removes the need for `x25_list_lock` to guard the `neighbour` field:

```c
restart:
    write_lock_bh(&x25_list_lock);
    sk_for_each(s, &x25_list) {
        sock_hold(s);
        write_unlock_bh(&x25_list_lock);
        lock_sock(s);
        if (x25_sk(s)->neighbour == nb)     /* check under socket lock */
            x25_disconnect(s, ENETUNREACH, 0, 0);
        release_sock(s);
        sock_put(s);
        goto restart;
    }
    write_unlock_bh(&x25_list_lock);
```

With `neighbour` now read and written exclusively under `lock_sock`, the `read_lock_bh`/`write_lock_bh` around `x25->neighbour = NULL` in `x25_disconnect` and `x25_connect` become unnecessary and should be removed.

This recommendation supersedes SYNC-001 (the SYNC-001 fix is a prerequisite interim measure; SYNC-006 is the proper long-term solution). Note that the `x25_kill_by_neigh` loop above processes all sockets, not just those matching `nb`, so it is slightly less efficient than the current approach, but the correctness gain and the removal of the dual-role invariant justify it. Alternatively, combine with SYNC-003's `sock_hold`-before-drop pattern to only hold each socket lock once.

**Estimated lines changed:** ~20 (`x25_kill_by_neigh` restructure; remove list-lock pairs from `x25_disconnect` and `x25_connect`)

**Affected functions:** `x25_kill_by_neigh` (`af_x25.c`), `x25_disconnect` (`x25_subr.c`), `x25_connect` (`af_x25.c`)

---

## Priority 4 — Low: Modernisation

### SYNC-007 — Convert all four global list `rwlock_t` instances to RCU

**Summary:** `x25_list_lock`, `x25_neigh_list_lock`, `x25_route_list_lock`, and `x25_forward_list_lock` are all `rwlock_t` (reader-writer spinlocks). In modern Linux kernel code this pattern is superseded by RCU for read-mostly linked lists. RCU allows completely lockless, non-spinning reads on the data path — significant at networking speeds.

**Why this matters:** The four lists are read on every inbound frame and on every outbound connect. Each `read_lock_bh` / `read_unlock_bh` pair involves a shared atomic counter and memory barriers. Under RCU, `rcu_read_lock()` is a preemption counter increment with no memory barrier on non-preemptible kernels, and `rcu_read_unlock()` is the corresponding decrement — effectively free.

**Proposed change per lock:**

Writers:
```c
/* Before: */
write_lock_bh(&x25_route_list_lock);
list_add(&rt->node, &x25_route_list);
write_unlock_bh(&x25_route_list_lock);

/* After: */
spin_lock_bh(&x25_route_list_lock);     /* spinlock_t replaces rwlock_t */
list_add_rcu(&rt->node, &x25_route_list);
spin_unlock_bh(&x25_route_list_lock);
```

Readers:
```c
/* Before: */
read_lock_bh(&x25_route_list_lock);
list_for_each_entry(rt, &x25_route_list, node) { ... }
read_unlock_bh(&x25_route_list_lock);

/* After: */
rcu_read_lock();
list_for_each_entry_rcu(rt, &x25_route_list, node) { ... }
rcu_read_unlock();
```

Deletions require `synchronize_rcu()` (or `call_rcu()`) after removal from the list and before freeing the object, unless the object is already protected by a refcount and freed via `kfree_rcu`.

The four locks share the same conversion pattern, so they can be done together with a consistent approach:

| Lock | Writers | Readers |
|---|---|---|
| `x25_list_lock` | `spin_lock_bh` + `list_add/del_rcu` | `rcu_read_lock` + `sk_for_each_rcu` |
| `x25_neigh_list_lock` | `spin_lock_bh` + `list_add/del_rcu` | `rcu_read_lock` + `list_for_each_entry_rcu` |
| `x25_route_list_lock` | `spin_lock_bh` + `list_add/del_rcu` | `rcu_read_lock` + `list_for_each_entry_rcu` |
| `x25_forward_list_lock` | `spin_lock_bh` + `list_add/del_rcu` | `rcu_read_lock` + `list_for_each_entry_rcu` |

`x25_neigh` and `x25_route` already use `refcount_t`, which is the required lifetime mechanism for RCU-protected objects.

**Estimated lines changed:** ~150 across `af_x25.c`, `x25_link.c`, `x25_route.c`, `x25_forward.c`, `x25_subr.c`, `x25_proc.c`

**Affected functions:** All list insert/remove/lookup functions and `/proc` sequence iterators across the six files above.

---

## Summary Table

| ID | Category | Impact | Fix Complexity | Lines | Files |
|---|---|---|---|---|---|
| SYNC-001 | Data race (write under read lock) | Critical | Trivial | 4 | `x25_subr.c`, `af_x25.c` |
| SYNC-002 | TOCTOU | High | Simple | ~12 | `x25_forward.c` |
| SYNC-003 | Race (unsafe iteration) | High | Simple | ~8 | `af_x25.c` |
| SYNC-004 | TOCTOU (LCI allocation) | High | Moderate | ~25 | `af_x25.c` |
| SYNC-005 | Missing lock (`x25_getname`) | Medium | Simple | 4 | `af_x25.c` |
| SYNC-006 | Design (dual-role lock) | Medium | Moderate | ~20 | `af_x25.c`, `x25_subr.c` |
| SYNC-007 | Modernisation (RCU) | Low/Perf | Large | ~150 | 6 files |

### Recommended implementation order

1. **SYNC-001** immediately — it is a four-line fix for a confirmed data race.
2. **SYNC-002** and **SYNC-003** together — both are small, independent fixes for races in unrelated files.
3. **SYNC-004** — moderate restructuring; can be done independently of the above.
4. **SYNC-005** — simple addition of `lock_sock` to an existing syscall handler.
5. **SYNC-006** — implement after SYNC-001 and SYNC-003 are in place, as it restructures the same functions.
6. **SYNC-007** — undertake as a separate modernisation task once the correctness fixes are stable.
