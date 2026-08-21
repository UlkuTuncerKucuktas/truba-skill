---
name: lustre
description: Working with the Lustre parallel filesystem — file layouts and striping (stripe_count, stripe_size, OST pools, stripe_offset), composite layouts (PFL) and Data-on-MDT, the lfs and lctl command families and how to parse their output, LDLM locking and what an open() actually costs, client RPC/readahead/grant tunables, client and server caching, and benchmarking Lustre I/O honestly from an unprivileged account. Use for any task touching lfs setstripe/getstripe/migrate, OSTs, MDTs, DoM, stripe tuning, or Lustre performance measurement.
---

# Lustre

Facts marked **[v]** were verified by running the command on a production
Lustre **2.15.3** filesystem (48 OSTs, 4 MDTs, two 24-OST pools). Measured
values are illustrative of one system, not universal — read them back on yours.

## Architecture

| Component | Role |
|---|---|
| MDS / **MDT** | metadata: namespace, inodes, layouts, locks. Can also hold small file *data* (see DoM). |
| OSS / **OST** | bulk file data as objects |
| **LOV** | client abstraction over all OSCs |
| **OSC** | client↔OST connection — **one OSC per OST** **[v]** 48 |
| **LMV** | client abstraction over all MDCs |
| **MDC** | client↔MDT connection — **one MDC per MDT** **[v]** 4 |

The one-OSC-per-OST fact is what makes stripe count a *concurrency* decision,
not just a placement one — see tunables below.

## Layouts and striping

A layout says where a file's data lives. It is chosen when the file is created
and cannot be changed by writing to it.

```bash
lfs setstripe -c 8 -S 1M DIR        # 8 objects, 1 MiB per stripe
lfs setstripe -c -1 DIR             # every OST in the applicable pool
lfs setstripe -i 8 -c 2 DIR         # start at OST index 8
lfs setstripe -o 0,1,4,5 DIR        # exact OST indices; implies -c 4
lfs setstripe -p flash DIR          # restrict to a pool
lfs getstripe -d DIR                # a directory's default layout
lfs getstripe FILE                  # what was actually granted
```

- `stripe_count` is the number of OST objects. `-1` means *all OSTs in the
  applicable pool*, which is **not** necessarily all OSTs in the filesystem.
  **[v]** on a directory defaulting to a 24-OST pool, `-c -1` granted **24** of
  48. That is the pool size, not degradation.
- `stripe_size` is bytes written to one object before moving to the next.
  Default 1 MiB. Must be a multiple of 64 KiB.
- `-i` sets the first OST; subsequent stripes walk pool order. **[v]** `-i 8 -c 2`
  landed on 8 and 9.
- `-o` names exact indices and sets the count from the list length. **[v]** exact.

**Objects are allocated eagerly for a plain layout, regardless of file size.**
**[v]** A 64 KiB file created with `-c 8` carries eight OST objects, seven of
which hold no data.

So two different quantities matter and they are routinely conflated:

- **objects allocated** = the granted `stripe_count`. Every one costs a lock at
  open (see Locking).
- **objects holding data** = `min(stripe_count, ceil(size / stripe_size))`.

A file smaller than `stripe_size` occupies exactly one object no matter how wide
its layout is. Benchmarks that vary only the first quantity cannot show a data
placement effect — but they *do* show a real open-time cost.

## Composite layouts: PFL and DoM

A composite layout has multiple **components**, each covering a byte **extent**
with its own striping.

```bash
# PFL: widen as the file grows
lfs setstripe -E 4M -c 1 -E 64M -c 4 -E -1 -c 32 DIR
# DoM: first component lives on the MDT
lfs setstripe -E 1M -L mdt -E -1 -c 1 -S 1M DIR
```

**PFL components are instantiated lazily** — objects for a later extent are
allocated only when the file grows into it. This is the opposite of plain
striping's eager allocation, so the two have genuinely different cost models.
`lcme_flags: init` marks an instantiated component.

### Data-on-MDT

The first component has `-L mdt` and its data lives on the MDT. **[v]** a small
DoM file holds **zero** OST objects.

What it buys is **RPC count**: the client gets "the file attributes, lock, and
data returned with a single RPC" instead of open/lock on the MDT, a glimpse to
the OST, an extent lock, then the read. The design docs model small-file write
cost dropping from `5185 + 4N` µs to `3056 + 2N` µs.

Two *independent* limits, and confusing them is the classic DoM mistake:

1. **The storage limit** is `lod.*.dom_stripesize` on the MDS. **[v]** it is not
   readable from a client. A request above it is **silently truncated, not
   refused** — **[v]** `-E 4M -L mdt` was granted as `e_end: 1048576` with no
   error or warning. Always read the granted extent back.
2. **The latency limit** is the reply buffer the data rides in. Past it the file
   is still on the MDT, still charged to metadata capacity, but needs a second
   RPC to read. `mdc.*.mdc_dom_min_repsize` (**[v]** 8192; note the `mdc_`
   prefix, which a naive glob misses) is a **floor the client grows from**, not a
   ceiling — so the parameter a user can read is not the one that decides whether
   DoM pays off.

MDT capacity is the scarce resource: **[v]** on the test system the four MDTs
total ~10.7 TiB against ~3.8 PiB of OST — **0.27 %**. Putting files on the MDT
spends the most limited space in the filesystem.

### Parsing composite layouts — the trap

```bash
lfs getstripe --component-count FILE   # 0 = plain layout, >0 = composite  [v]
lfs getstripe -c FILE                  # NOT a total
```

**[v]** On a composite file `-c` reports the *last instantiated component's*
count: **0** for a small DoM file (the MDT component has `stripe_count 0`), and
**4** for the same layout once the file grew into its 4-stripe OST component.

Never scrape the full `lfs getstripe` text for "the biggest number" — it contains
`lcme_extent.e_end: 1048576`, and that is a byte offset, not an object count.
Discriminate on `--component-count` first, then walk components.

## OST pools

```bash
lfs pool_list <fsname>              # list pools
lfs pool_list <fsname>.<pool>       # members, as hex UUIDs
```

Pool membership need not be a contiguous index range. **[v]** on the test system
two pools interleave in pairs: one holds 0,1,4,5,8,9,… and the other 2,3,6,7,…
so `-o 0,1,2,3` straddles both tiers. `lfs pool_list` prints hex UUIDs while
`-o` takes decimal — convert.

A directory's default pool is inherited by files created in it, and `-c -1` then
means "all OSTs *in that pool*".

## Locking — why width costs something at open

Lustre uses a distributed lock manager (LDLM). The essential fact for
performance work:

> **The client enqueues a lock for each stripe of the file, to the respective
> OST.**

So an open pays lock enqueues proportional to the number of objects, plus a
glimpse to learn the file size. That is a per-object constant paid whether or
not the object holds data — the mechanism behind "wide layouts are slower for
small files".

DoM instead takes "a single lock covering the entire data range on the MDT
object", which is why its open is cheap and its read is nearly free.

Clients cache **under lock**. Two consequences that bite benchmarks:

- Releasing the write lock when a file is closed **surrenders the client's
  cached pages**. A freshly written, closed file is already cold on the client.
- Any reopen takes a lock. **Reopening a DoM file to "evict" it destroys the
  inlining you were trying to measure** — including opening it write-only. Do
  not write an `evict()` that opens the file.

`ldlm.namespaces.*.lru_size` would drop locks properly but **[v]** is not
writable by an unprivileged user.

## Client tunables

```bash
lctl get_param osc.*.max_rpcs_in_flight mdc.*.max_rpcs_in_flight
lctl get_param llite.*.max_read_ahead_mb llite.*.statahead_max
lctl list_param osc.*.rpc_stats
```

| Parameter | Scope | **[v]** value | Why it matters |
|---|---|---|---|
| `osc.*.max_rpcs_in_flight` | **per OSC = per OST** | 8 | **Total client read concurrency ≈ value × stripe_count.** 1 object → 8 RPCs; 24 objects → 192. This is *the* mechanism by which width buys parallelism. |
| `osc.*.max_pages_per_rpc` | per OSC | 4096 | 4096 × 4 KiB = 16 MiB per RPC |
| `osc.*.max_dirty_mb` | per OSC | 2000 | write buffering before blocking |
| `osc.*.cur_grant_bytes` | per OSC | ~1.75 GiB | space the server has granted for un-acked writes |
| `mdc.*.max_rpcs_in_flight` | per MDC | 8 | metadata concurrency per MDT |
| `llite.*.max_read_ahead_mb` | per mount | 1024 | client readahead |
| `llite.*.max_read_ahead_per_file_mb` | per mount | **256** | **a whole test file may be prefetched on first touch** |
| `llite.*.statahead_max` | per mount | 32 | prefetches attributes during directory traversal |

**[v]** All of these are **read-only to an unprivileged user** (`Permission
denied` on `lctl set_param`). Treat them as fixed environmental constants and
record them alongside any measurement; do not plan to tune them.

## Caching, and defeating it without root

Two separate caches, needing two separate defences:

- **Client** — held under lock. Close the file (release the write lock) and the
  pages are gone. Do not "evict" by reopening.
- **Server (OSS page cache)** — an unprivileged user cannot drop it. The only
  levers are volume and ordering: write sequentially and read regions with a
  large volume of intervening writes behind them, advance the read region
  between repeats, and bound the claim honestly ("no effect with N GiB of
  intervening writes"). `obdfilter.*.readcache_max_filesize` on the OSS makes
  large files bypass the server cache if set, but is not client-readable.

A read that returns at memory-and-network speed rather than device speed is a
cache hit, not a result. If no device was the limit, spreading data over more
devices cannot help — a null under those conditions is a property of the
measurement.

## Changing a layout after creation

A layout is fixed at creation and writes cannot alter it. Lustre allocates it at
first open, so **the file's eventual size cannot inform the decision** — this is
why DoM residency must be chosen in advance.

It *can* be changed afterwards, by copying every byte:

```bash
lfs migrate -c 4 FILE                        # [v] 8 stripes -> 4
lfs migrate -E 1M -L mdt -E -1 -c 1 FILE     # [v] plain OST file -> DoM
```

`lfs migrate` rewrites the data into a new layout. So layout is a placement
decision revisable only at the cost of a full data copy — not a runtime knob an
adaptive tuner can turn cheaply.

## Gotchas

| Trap | Reality |
|---|---|
| `lfs setstripe` on a path that doesn't exist | **[v]** creates a zero-length *file* at that path. `mkdir` first. |
| Assuming the request was granted | Requests above a limit are silently truncated. Read back, always. |
| `-c -1` means "all OSTs" | Means all OSTs **in the pool**. **[v]** 24 of 48. |
| `lfs getstripe -c` on a composite | Not a total — the last instantiated component. **[v]** |
| Scraping getstripe text for a count | Picks up `e_end` byte offsets. Use `--component-count`. |
| `-o 0,1,2,3` picks the first four | Pool indices interleave. **[v]** Read `lfs pool_list`. |
| Evicting cache by reopening the file | Takes a lock; destroys DoM inlining. Just close after writing. |
| Tuning readahead / RPCs for a benchmark | **[v]** read-only unprivileged. Record them instead. |
| Wide stripe always faster | Costs one lock per object at open, paid even for empty objects. **[v]** |
| Comparing arms that differ only in allocated objects | Tests open cost, not placement. State which. |

## Admin-only

**[v]** `lfs setdirstripe` (DNE striped directories) returns `Operation not
permitted` for a normal user, as does `lctl set_param` on `llite.*` and
`ldlm.*`. Metadata-striping and client-tunable experiments need administrator
access; layout, pool and DoM experiments do not.

## Useful commands

```bash
lfs df /mnt ; lfs df -i /mnt        # per-target space / inodes
lfs osts /mnt ; lfs mdts /mnt
lfs quota -h -u $USER /mnt
lfs find /mnt --stripe-count +8     # find files by layout
lfs getstripe --component-count F
lctl get_param -n version
lctl list_param osc.*.rpc_stats     # [v] present; llite.*.stats was not
```

**[v]** Stats cannot be cleared without privileges, so take deltas, and close
the counter window exactly where the measured phase ends.
