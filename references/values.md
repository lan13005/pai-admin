# Recorded values

Snapshots of inventory and QOS config for quick citation. **Re-verify before acting on them** with
`gfree`, `scontrol show partition`, `sacctmgr show qos`, and `/etc/slurm/job_submit.lua`.

## GPU types

| GPU Model | Mem | Cores/GPU | CPU-Mem/GPU | Partition |
|-----------|-----|-----------|-------------|-----------|
| H200 | 141 GB | 8 | 185 GB | ailab / ailab-p (restricted) |
| H100 | 80 GB | 12 | 120 GB | pli* (restricted) |
| A100 80GB | 80 GB | 12 | 240 GB | general (`--constraint=gpu80`) |
| A100 40GB | 40 GB | 64 | 360 GB | general (`--constraint="nomig&gpu40"`) |
| MIG A100 40GB | 40 GB | 6 | 120 GB | general (`--constraint="intel&gpu40"`) |
| MIG A100 10GB | 10 GB | 1 | 32 GB | general (`--constraint=mig`) |

## Partition inventory

All GPU partitions here bill `CPU=1.0, Mem=0.1G, GRES/gpu=12`. The `pli*` partitions overlap heavily —
the GPU counts below are not additive.

| Partition | Nodes | GPUs | MaxTime | Notes |
|-----------|-------|------|---------|-------|
| `ailab` | 18 (`della-i19g[1-3]` … `della-i24g[1-3]`) | 144 × H200 (8/node) | 15 d | 1.5 TB, 64 CPU/node; partition QOS `ailab` |
| `ailab-p` | same 18 nodes as `ailab` | same H200 pool | 15 d | project-account partition; partition QOS `ailab-p` |
| `pli` | 38 (`della-j*`, `della-k*`) | 304 × H100 (8/node) | 15 d (filter caps 3 d) | 1 TB, 96 CPU/node; campus tier via `pli-low` |
| `pli-c` | 42 | 336 × H100 | 3 d | superset of `pli`; partition QOS `part-pli-c` (64 GPU/user) |
| `pli-p` | 42 | 336 × H100 | 15 d (filter caps 3 d) | same nodes as `pli-c` |
| `pli-lc` | 9 (`della-j[13-15]g[1-3]`) | 72 × H100 | 3 d | subset of `pli` |

`job_submit.lua` rejects any job over `VLONG_MINS` (≈ 6 days) before it reaches the partition, and PLI partitions are capped at 3 days regardless of the partition `MaxTime` above.
Quote the submit filter, not `MaxTime`, when telling a user how long they can ask for —
[submit-restrictions.md](submit-restrictions.md).

## Walltime → `gpu-*` (ailab / general GPU flow)

The bins are Lua constants, the authoritative copy is in
[submit-restrictions.md](submit-restrictions.md#walltime-bins-general-gpu-flow). Why the tier matters for
queue order: [priority.md](priority.md).

## QOS: `gpu-*`

Assigned from walltime on `ailab*` and general GPU partitions. On `ailab*` the `MaxGPUs/User` column
is overridden by the partition QOS below.

| QOS | Priority | MaxGPUs/User | MaxNodes/User | MaxJobs/User | Notes |
|-----|----------|--------------|---------------|--------------|-------|
| `gpu-test` | 8000 | — | — | 3 | Quick testing |
| `gpu-short` | 5000 | 44 | — | 44 | Default short GPU |
| `gpu-medium` | 2000 | 20 | 24 | 24 | Group limit: 160 GPUs |
| `gpu-long` | 1000 | 16 | 16 | 10 | Long-running |

## QOS: partition QOS

These **override** the job QOS limits (no QOS here carries `OverPartQOS`), while priority still comes
from the job QOS — see [priority.md](priority.md#how-the-two-qos-interact). So `MaxGPUs/User` below is
the number that actually binds on those partitions.

| QOS | On partition | Priority (not used for job priority) | MaxGPUs/User | Notes |
|-----|--------------|-------------------|--------------|-------|
| `ailab` | `ailab` | 0 | **16** | Binds for every `gpu-*` tier |
| `ailab-p` | `ailab-p` | 8000 | **24** | Group limit: 48 GPUs |
| `part-pli-c` | `pli-c` | 0 | **64** | — |

`pli`, `pli-lc`, and `pli-p` have no partition QOS, so their `pli-*` limits apply directly.

## QOS: `pli-*`

Assigned from the **partition** (see [priority.md](priority.md)), and must be granted on the user's
account ([accounts.md](accounts.md)).

| QOS | Priority | MaxGPUs/User | Group GPUs | Per account | Notes |
|-----|----------|--------------|------------|-------------|-------|
| `pli-short` | 5000 | 64 | — | — | also max 10 nodes; `pli-c` |
| `pli-medium` | 3000 | 32 | 176 | — | max 8 nodes; `pli-c` |
| `pli-cp` | 5050 | 8 | — | — | `pli-c` debugging, ≤ 8 h |
| `pli-high` | 6000 | 128 | 128 | 128 | `pli-p` |
| `pli-lc` | 1500 | 16 | 32 | 16 | `pli-lc`; max 16 nodes |
| `pli-low` | 0 | 16 | 48 | — | `pli` campus tier |
| `pli-long` | 0 | 8 | 8 | — | legacy tier |
