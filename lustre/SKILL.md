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

So the client pays lock enqueues proportional to the number of objects, plus a
glimpse to learn the file size — a per-object constant paid whether or not the
object holds data. This is the mechanism behind "wide layouts are slower for
small files".

**[v] It is not charged to `open()`.** Measured cold from a second node on
200 x 256 KiB files (every file smaller than one stripe, so all data sits on one
object regardless of width):

| width | total | `open` | first read |
|---|---|---|---|
| 1 | 1420 us | 81 us | 1332 us |
| 8 | 1476 us | 82 us | 1370 us |
| 24 | 1638 us | 98 us | 1481 us |

Width costs roughly **9.5 us per allocated object**, and most of it lands in the
**first read**, not the open. Lustre's `open` returns the layout;
the extent locks are enqueued when I/O actually happens. So a harness that times
only `open()` will miss the effect almost entirely — decompose into
open / first-read / rest-read.

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

**[v] Measured, one client, 100 x 20 MiB working set, `-c 1`:**

| condition | aggregate |
|---|---|
| read immediately after writing | **39.5 GiB/s** |
| after writing 16 GiB more | 24.5 GiB/s |
| after writing 64 GiB more | 7.3 GiB/s |
| `O_DIRECT`, no filler | 5.0 GiB/s |

The naive number is **8x** the honest one. `O_DIRECT` **[v] works on Lustre** and
is a far cheaper client-cache bypass than writing 64 GiB per cell — but it also
disables readahead, so it measures a different I/O path, not just a colder one.
Use it as a control arm, not as the headline configuration.

### The writer must flush, or you measure lock revocation

Reading from a second node is necessary but **not sufficient**. The writing node
still holds the write locks, so the reader's request forces blocking callbacks
back to it. That inflates every number — and, fatally, it inflates them
*unequally*:

```bash
lfs data_version -w FILE      # writer flush: push data AND release the write lock
```

**[v]** 200 files per arm, written on one node, read cold from another:

| arm | no flush | flushed | inflation |
|---|---|---|---|
| DoM 64 KiB | 883 us | **212 us** | **4.17x** |
| OST 64 KiB | 1054 us | 637 us | 1.65x |
| DoM 4 KiB | 388 us | 156 us | 2.49x |
| OST 4 KiB | 507 us | 277 us | 1.83x |

Skipping the flush costs DoM 4.2x and OST 1.7x, so it does not cancel in a
ratio — it **destroys the DoM signal specifically**, turning a 3x speedup into a
1.2x one. Any DoM comparison without a writer flush is measuring the wrong thing.

`lfs data_version -w` is the reliable interface. The underlying
`LL_IOC_DATA_VERSION` ioctl is **[v]** not straightforwardly callable from Python
(the obvious `_IOR('f', 249, 16)` encoding returns `ENOTTY`); shelling out per
file is fine because it belongs in the write phase, outside any timed region.

### DoM inlining has a visible cliff

**[v]** Cross-node, flushed, 200 files, T=1 — the `first_read / open` split is the
discriminator:

| size | DoM total | DoM open | DoM first read | OST total | OST/DoM |
|---|---|---|---|---|---|
| 4 KiB | 155 us | 145 | **5.2** | 277 us | 1.78x |
| 16 KiB | 168 us | 157 | **6.2** | 618 us | **3.67x** |
| 64 KiB | 212 us | 197 | **9.4** | 637 us | **3.01x** |
| 128 KiB | 599 us | 66 | **526** | 651 us | 1.09x |

Below the cliff the data arrives *with the open* — the read is a few
microseconds of memcpy and `open` carries the cost. Above it, `open` falls back
to a normal ~66 us and the read costs a full round trip. The benefit collapses
from 3x to 1.1x between 64 and 128 KiB.

`mdc.*.mdc_dom_min_repsize` (**[v]** 8192) is confirmed to be a **floor, not a
ceiling** — inlining still worked at 64 KiB, eight times that value. The client
grows the reply buffer (`cl_dom_min_inline_repsize` in `mdc_intent_open_pack`),
so the real boundary must be measured, not read from a parameter.

**Client caching cannot be defeated on the node that wrote the file.** Closing
the file releases the write lock, but the client keeps pages and locks warm
enough to erase the effect entirely. **[v]** measured on 500 x 64 KiB files:
DoM 78.1 us vs OST 77.8 us — *no difference at all* — when the reader ran on the
writer's node. The same comparison from a **second node in the same job** gave
DoM 908 us vs OST 1056 us. The reliable recipe for a cold small-file read is:

```bash
#SBATCH -N 2
NODES=($(scontrol show hostnames $SLURM_JOB_NODELIST))
srun -N1 -n1 -w ${NODES[0]} python3 bench.py write
srun -N1 -n1 -w ${NODES[1]} python3 bench.py read     # never saw these files
```

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


## Benchmarking Lustre from Python

**[v]** Measured on a 2.15.3 client, Python 3.9:

- `os.pread`, `os.preadv`, `os.pwrite`, `os.posix_fadvise`, `os.sched_setaffinity`
  and `os.O_DIRECT` are all available. `perf_counter_ns` resolves to ~85 ns
  back-to-back, which is negligible against microsecond-scale I/O.
- **The interpreter adds roughly 20-27 us per file** (open + reads + close),
  measured against fully cached node-local files. That is ~2 % of a millisecond
  Lustre read and **~30 % of a small DoM read** — so it is negligible for
  striping work on files of 256 KiB and up, and *not* negligible in the
  small-file regime. Measure this floor on the same node in the same run and
  report it.
- The floor is **flat in thread count** (23.7 / 28.8 / 28.3 / 26.9 us at
  T = 1 / 4 / 16 / 32), so the GIL does not degrade with concurrency here.
- **`ThreadPoolExecutor` spawns workers lazily, and the cost scales with the
  pool size** — **[v]** 389 / 610 / 1213 / 2153 us to create and spawn 1 / 4 /
  16 / 32 workers. Timing a region that includes pool construction inflated the
  measurement by **1.29x at T=32 versus 1.05x at T=1**, manufacturing a penalty
  that looks exactly like a client-side concurrency ceiling. Build the pool and
  drive every worker through a `threading.Barrier` *before* starting the clock.


## Width: cost, benefit, and the interior optimum

**[v]** Cross-node, writer-flushed, median per-file latency relative to `-c 1`:

| file size | spans | w=4 | w=8 | w=24 |
|---|---|---|---|---|
| 256 KiB (T=1) | 1 object | 1.12x | 1.12x | **1.30x** |
| 1 MiB (T=1) | 1 object | 1.09x | 1.06x | 1.22x |
| 4 MiB (T=1) | 4 objects | **0.88x** | **0.88x** | 0.91x |

With a 1 MiB stripe, a file's data occupies `ceil(size / stripe_size)` objects
however wide the layout is. Below that bound width is pure cost; at or above it
width buys real parallelism. A 4 MiB file spans four objects, and the optimum
sits at **w=4-8 — neither 1 nor the maximum** — because w=24 adds twenty empty
objects that still cost locks.

At high concurrency (T=32) the differences compress toward 1.0x: per-file
latency is dominated by queueing rather than layout. **Record aggregate
throughput as well as per-file latency** — at T=32 the median latency rose while
the arms became indistinguishable, and the two numbers answer different
questions.


## The read path, stage by stage

**[v]** Cold, cross-node, writer-flushed, T=1, medians in microseconds. Each stage
measured on a disjoint set of 100 files so every one is a first touch. `BRW` is
the delta in OST bulk-read RPCs per file, from `osc.*.rpc_stats`.

| arm | `open` | `fstat` | read 1 B | read all | BRW/file |
|---|---|---|---|---|---|
| DoM 4 KiB | 120.0 | **3.7** | 3.7 | 9.7 | **0.00** |
| DoM 64 KiB | 208.7 | **3.8** | 3.7 | 14.8 | **0.00** |
| OST 4 KiB `c=1` | 59.9 | 51.1 | 197.4 | 195.0 | 3.0 |
| OST 64 KiB `c=1` | 63.1 | 49.2 | 190.6 | 557.8 | 3.0 |
| OST 64 KiB `c=24` | 81.9 | **131.6** | 189.8 | 732.1 | 3.0 |
| OST 4 MiB `c=1` | 62.5 | 46.7 | 189.7 | 3025.8 | 6.0 |
| OST 4 MiB `c=4` | 64.5 | 57.7 | 191.5 | **2498.2** | 12.0 |
| OST 4 MiB `c=24` | 82.2 | **130.9** | 195.1 | 2672.7 | 12.0 |

### What each stage is

**`open`** — one `MDS_OPEN` intent RPC to the MDT via the MDC. It returns the
inode attributes, the open handle, and the **layout** (which OSTs hold the
objects). It does *not* return the file size for an OST file. Cost grows weakly
with stripe count — **[v] ~0.87 us per object** — because the layout itself is
bigger.

**`fstat` — the size query, and this is where width is charged.** The
authoritative file size lives on the OSTs: each object knows its own length, and
the MDT does not. So the client must **glimpse every stripe** and sum the
results. **[v]** the cost scales with stripe count:

| stripes | `fstat` |
|---|---|
| 1 | ~47-51 us |
| 4 | ~58 us |
| 24 | **~131 us** |

That is **[v] ~3.65 us per additional object**, paid whether or not the object
holds any data. Together with the layout term, an object that will never hold a
byte still costs about **4.5 us** on every cold open-and-stat.

**read of one byte** — **[v] flat at ~190 us regardless of stripe count.** A read
of `[0,1)` needs an extent lock covering only that range, so it touches exactly
one object. This is the fixed price of talking to an OST at all: LDLM extent
lock enqueue plus one bulk RPC. **Locks are per extent, not per file** — which is
why the width penalty appears in the size query, not in the data path.

**read all** — bulk transfer, LNet RDMA, `max_pages_per_rpc` (**[v]** 4096 pages
= 16 MiB) per BRW RPC. This is the stage width actually helps: 4 MiB over four
objects (2498 us) beats the same file on one (3026 us), while spreading it over
24 (2673 us) is worse again because twenty of them are empty.

**DoM** short-circuits the whole sequence. The data rides back in the `open`
reply, so `open` grows with file size (120 -> 209 us from 4 to 64 KiB) while
`fstat` costs **3.7 us** (the MDT already told the client the size) and the read
is a memcpy. **[v] Zero OST bulk RPCs** — proof the data never reaches an OST.

### Three small-I/O regimes, not one

**[v]** `osc.*.short_io_bytes` = **16384**. Reads and writes at or below that size
are carried inline in the RPC instead of by bulk RDMA — an OST-side inline path
*independent of DoM*. That creates three regimes:

| size | OST path | DoM advantage |
|---|---|---|
| <= 16 KiB | short I/O, inline, no RDMA | smaller — OST is inlining too (**[v]** 1.78x at 4 KiB) |
| 16 KiB .. inline cliff | bulk RDMA | largest (**[v]** 3.0-3.7x) |
| above the cliff | bulk RDMA | gone (**[v]** 1.09x at 128 KiB) |

A DoM evaluation that samples only very small files understates the benefit,
because `short_io` is competing with it there.

### The cost model this implies

```
cold read latency  ~  open(layout)      + 0.87 us x objects_allocated
                    + glimpse(size)     + 3.65 us x objects_allocated
                    + extent lock + BRW ~ 190 us fixed
                    + bulk transfer     / objects_with_data
```
where `objects_with_data = min(stripe_count, ceil(size / stripe_size))`. Objects
beyond that bound contribute only to the first two terms — they are pure cost.
