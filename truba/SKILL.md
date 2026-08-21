---
name: truba
description: Working on TRUBA (TÜBİTAK ULAKBİM Turkish national HPC) — connecting to arf-ui/cuda-ui login nodes, writing or debugging SLURM batch scripts, choosing a partition/queue, the /arf Lustre filesystem (striping, pools, quotas, Data-on-MDT), module/Python setup, or diagnosing job failures and rejections. Use whenever a task touches TRUBA, /arf, cuda-ui, arf-ui, orfoz, barbun, hamsi, kolyoz, palamut, akya, or a TRUBA sbatch/srun invocation.
---

# TRUBA

Turkish national HPC. Two separate SLURM clusters sharing one Lustre filesystem.
Facts marked **[v]** were verified on the machine; the rest come from
docs.truba.gov.tr.

## Connect

Requires TRUBA OpenVPN to be up (all addresses are RFC1918).

```bash
ssh <username>@172.16.6.11    # arf-ui1  → cluster "arf"
ssh cuda-ui                  # 172.16.6.16 → cluster "cuda"
```

Other published services: Open OnDemand `172.16.6.20`, Grafana
`172.16.6.25:3000`.

**[v]** `arf-ui1..arf-ui5` = `172.16.6.11..15`, `cuda-ui` = `172.16.6.16`. Both
reachable directly from a VPN-connected laptop. `levrek1`, `palamut-ui`,
`sardalya1`, `sage-ui` appear in `/etc/hosts` but do **not** answer — do not
assume a hostname in that file is live.

Login nodes are for editing, compiling and short work only. Running heavy jobs
there gets the account **suspended**, not just the process killed.

## Which cluster has which partition

This is the single most common source of wasted time. Partitions do **not** span
clusters.

**`arf` (arf-ui1) — all CPU-only partitions live here** **[v]**

| Partition | Nodes | Cores/node | Mem | GPU | Max time | Min cores/job |
|---|---|---|---|---|---|---|
| `orfoz` (default) | 192 | 112 | 256 GB | — | 3 d | 56 |
| `hamsi` | 129 | 56 | 192 GB | — | 3 d | 28 |
| `barbun` | 120 | 40 | 384 GB | — | 3 d | 20 |
| `smp` | 1 | 224 | 4 TB | — | 3 d | — |
| `debug` | 273 | 40+ | mixed | some | **4 h** | — |
| `akya-cuda` | 16 | 40 | 384 GB | 4× V100 | 3 d | 10 + 1 GPU |
| `barbun-cuda` | 24 | 40 | 384 GB | 2× P100 | 3 d | 20 + 1 GPU |

**`cuda` (cuda-ui)** **[v]**

| Partition | Nodes | Cores/node | GPU | Max time | Rule |
|---|---|---|---|---|---|
| `kolyoz-cuda` | 59 | 64 | 4× H100 or H200 | 3 d | 16 cores per GPU; pick with `-C H100` / `-C H200` |
| `palamut-cuda` | 6 | 128 | 8× A100 | 3 d | 16 cores per GPU |

`kolyoz-cuda` and `palamut-cuda` are restricted to infrastructure projects and
contracted users; everyone else uses `barbun-cuda` / `akya-cuda` on `arf`.

**If the job needs no GPU, use `arf`.** `debug` (4 h, big, usually free) is the
right place to iterate; `barbun` for longer CPU runs.

## Submitting

```bash
#!/bin/bash
#SBATCH -p debug
#SBATCH -C barbun            # pick hardware inside the mixed debug partition
#SBATCH -A <username>          # REQUIRED
#SBATCH -J myjob
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 20                # >= the partition minimum
#SBATCH --time=0:30:00       # REQUIRED
#SBATCH --output=/arf/scratch/$USER/%x-%j.out
```

Hard rules, each of which shows up only as a rejection or a dead job:

- **`-A <username>` is required.** Omitting it is rejected.
- **`--time` is required.** Without it the job is killed after ~1 minute
  (`debug` has `DefaultTime=00:02:00` **[v]**).
- `--output` / `--error` must point under `/arf/scratch`.
- GPU partitions **reject jobs that do not request a GPU**. `--gres=gpu:N` is
  mandatory there, and the core:GPU ratio is enforced **per node** — so `-N` and
  `--ntasks` must make the arithmetic work. An `sbatch --wrap` carrying only
  `--cpus-per-task` and `--gres` gets rejected on an arithmetically valid request.
- Omitting `--gres` on the command line does **not** cancel a `--gres` line in
  the header. Pass `--gres=NONE`.
- `#SBATCH` lines are comments parsed before any shell runs, so **they cannot
  read environment variables**. Put site-varying geometry (core counts) on the
  command line, where it overrides the header.
- The working directory must be under `/arf/scratch`.
- Give a *realistic* `--time`. The scheduler backfills, so an honest short limit
  starts sooner than a padded one.
- Add `--no-requeue` for jobs without checkpointing; node failures otherwise
  restart them from the beginning.

Interactive:

```bash
srun -p debug -C barbun -N 1 -n 1 -c 20 -A <username> -J test --time=0:30:00 --pty /usr/bin/bash -i
```

Per-node core counts for the `-c` argument: orfoz 55, hamsi 54, barbun 20,
barbun-cuda 20 (+`--gres=gpu:1`), akya-cuda 10 (+`--gres=gpu:1`).

## Monitoring

```bash
lssrv        # TRUBA-specific: free/total cores + waiting jobs per partition. Check BEFORE choosing.
avci <term>  # TRUBA-specific: search modules ( -exact for exact names )
squeue -u $USER
sacct -X -u $USER --starttime=today -o JobID%12,JobName%20,Partition%12,State%18,Elapsed,ReqCPUS
sinfo -o "%P %C"
scontrol show partition debug
```

## Storage

| Path | Quota (documented) | Lifetime | Notes |
|---|---|---|---|
| `/arf/home/$USER` | 2 TB + 500K inodes | user-managed | not backed up |
| `/arf/scratch/$USER` | 2 TB + 500K inodes | **purged after ~30 days** | run jobs from here |
| `/tmp` | node-local NVMe | per job | kolyoz 7 TB; palamut `/localscratch` 12 TB; akya-cuda 1.4 TB |

**Nothing is backed up.** Scratch is swept periodically.

**[v]** `lfs quota -u $USER /arf` reports the default setting with no enforced
block or inode limit, and an account was observed at 910 GB / 1.75 M files —
well past the documented 500K inodes. Treat the documented quota as policy that
may be enforced later, not as a limit the filesystem will stop you at.

**No conda/miniconda/pip installs onto `/arf`.** Explicit policy: hundreds of
thousands of small files degrade the filesystem for everyone. Use modules
or an Apptainer container. Example scripts live in `/arf/sw/scripts/`.

**[v]** 136 modules available. Verified names — note the docs say
`comp/python/3.12.2`, which **does not exist**:

```
comp/python/3.12.0   comp/python/intelpython3   comp/python/miniconda3
apps/truba-ai/cpu-2024.0   apps/truba-ai/gpu-2024.0
lib/cuda/11.8   lib/cuda/12.4   lib/cuda/12.6
```

**[v]** The system interpreter on the login nodes is **Python 3.9.18** — code
meant to run without a module load must target 3.9 (no `match`, no `X | Y`
runtime annotations, no `tomllib`). A non-interactive `ssh host 'module avail'`
finds nothing until you source the init script:

```bash
source /etc/profile.d/modules.sh 2>/dev/null || source /usr/share/lmod/lmod/init/bash
```

## The Lustre filesystem (`/arf`)

**[v]** Lustre **2.15.3**, 13.8 PB, ~73 % used, **48 OSTs**, **4 MDTs**, mounted
identically on both clusters from the same MDS/OSS addresses.

**[v]** Two pools of 24 OSTs each, and **their indices interleave in pairs** —
they are not contiguous ranges. Verified full membership (decimal):

```
lustre1.flash : 0,1, 4,5, 8,9, 12,13, 16,17, 20,21, 24,25, 28,29, 32,33, 36,37, 40,41, 44,45
lustre1.disk  : 2,3, 6,7, 10,11, 14,15, 18,19, 22,23, 26,27, 30,31, 34,35, 38,39, 42,43, 46,47
```

So `lfs setstripe -o 0,1,2,3` straddles both tiers — indices 2 and 3 are
capacity disks. Read the real membership rather than assuming a range
(`lfs pool_list` prints hex UUIDs; `-o` takes decimal):

```bash
lfs pool_list lustre1.flash | tail -n +2 | sed 's/lustre1-OST//;s/_UUID//'
```

**[v]** `/arf/scratch` inherits `stripe_count 1, stripe_size 1M, pool flash`.
Because scratch is pinned to the 24-OST flash pool, **`-c -1` grants 24, not
48** — that is the pool size, not degradation.

```bash
lfs getstripe -d DIR              # default layout of a directory
lfs getstripe FILE                # granted layout — always read back, never assume
lfs setstripe -c 8 -S 1M DIR
lfs setstripe -o 0,1,4,5 DIR      # pin exact OST indices (implies stripe count)
lfs setstripe -E 1M -L mdt -E -1 -c 1 -S 1M DIR   # Data-on-MDT
lfs df /arf ; lfs osts /arf ; lfs mdts /arf
lfs quota -h -u $USER /arf
```

Gotchas that have each cost real measurements:

- **`setstripe` on a non-existent path creates a FILE.** `mkdir` first.
- A layout request above the MDT cap is **silently truncated, not refused** —
  read the granted value back.
- `lfs getstripe -c` on a **composite** (PFL/DoM) file prints the whole layout;
  taking the largest integer returns an extent boundary in bytes. Sum
  `--stripe-count` across components instead.
- **[v]** `lod.*.dom_stripesize` is an MDS-side parameter and is **not readable
  from a client**. `mdc.*.mdc_dom_min_repsize` reads 8192 on all four MDTs —
  note the `mdc_` prefix, which a naive glob misses. It is a *floor* the client
  grows from, not a ceiling.
- **[v]** Client readahead and statahead are on and are large enough to mask
  layout effects in any I/O benchmark. Capture all of these in provenance:
  `max_read_ahead_mb` **1024**, `max_read_ahead_per_file_mb` **256**,
  `statahead_max` **32**, `checksum_pages` **1**.
- An unprivileged user cannot drop an OSS cache and cannot write
  `ldlm.namespaces.*.lru_size`. Defeat server caching with volume and ordering
  (large intervening writes, advancing read regions), and bound the claim by how
  much was written.
- Lustre caches under lock: closing a freshly written file releases the write
  lock and surrenders the client's pages, so **a just-written file is already
  cold**. Do not "evict" by reopening it — any reopen, write-only included,
  takes a lock and destroys Data-on-MDT inlining.

## Common failures

| Symptom | Cause |
|---|---|
| Job dies after ~1 min | no `--time` |
| Rejected at submit | missing `-A`, cores below partition minimum, GPU partition without `--gres`, or wrong core:GPU ratio |
| `AssocGrpCpuLimit` | asked for more simultaneous cores than your allocation (PhD/academic ≈ 160; MSc 40; undergrad 4) |
| `AssociationJobLimit` / `AssocGrpCPUMinutesLimit` | core-hour quota exhausted — update publications on the portal, then mail trubadestek@tubitak.gov.tr |
| Job pending while nodes show IDLE | resources held for higher-priority or earlier long jobs; backfill needs an honest `--time` |
| Multi-node job fails instantly | ranks racing on a shared path — scope every path by job id **and** rank |
| `task/cgroup: plugin not compiled with hwloc support` | harmless warning, ignore |
| `Socket timed out` on squeue/sbatch | scheduler load, retry |
| Account suspended | ran heavy work on a login node |
| Partition "does not exist" | you are on the wrong cluster — see the table above |

Support: trubadestek@tubitak.gov.tr · Portal: https://portal.truba.gov.tr ·
Docs: https://docs.truba.gov.tr
