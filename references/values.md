# Recorded values

Snapshots of inventory and QOS config for quick citation. **Re-verify before acting on them** with
`gfree`, `scontrol show partition`, `sacctmgr show qos`, and `/etc/slurm/job_submit.lua`.

## GPU types

| GPU Model | Mem | Cores/GPU | CPU-Mem/GPU | Partition |
|-----------|-----|-----------|-------------|-----------|
| H200 | 141 GB | 8 | 185 GB | ailab / ailab-p (restricted) |
| H100 | 80 GB | 12 | 120 GB | pli (restricted) |
| A100 80GB | 80 GB | 12 | 240 GB | general (`--constraint=gpu80`) |
| A100 40GB | 40 GB | 64 | 360 GB | general (`--constraint="nomig&gpu40"`) |
| MIG A100 40GB | 40 GB | 6 | 120 GB | general (`--constraint="intel&gpu40"`) |
| MIG A100 10GB | 10 GB | 1 | 32 GB | general (`--constraint=mig`) |

## Partition inventory

| Partition | Nodes | GPUs | Notes |
|-----------|-------|------|-------|
| `ailab` | 18 (`della-i19g[1-3]` … `della-i24g[1-3]`) | 144 × H200 (8/node) | ~1.5 TB/node, 64 CPU/node; billing CPU=1.0, Mem=0.1G, GPU=12 |
| `ailab-p` | same node family as `ailab` | same H200 pool | project-account partition; confirm live |
| `pli` | — | 336 × H100 | QOS tiers below |

`job_submit.lua` rejects any job over `VLONG_MINS` (≈ 6 days) before it reaches the partition, and PLI partitions are capped at 3 days.
Quote the submit filter, not `MaxTime`, when telling a user how long they can ask for —
[submit-restrictions.md](submit-restrictions.md).

## Walltime → `gpu-*` (ailab / general GPU flow)

The bins are Lua constants, the authoritative copy is in
[submit-restrictions.md](submit-restrictions.md#walltime-bins-general-gpu-flow). Why the tier matters for
queue order: [priority.md](priority.md).

## QOS: `gpu-*`

| QOS | Priority | MaxGPUs/User | MaxNodes/User | Notes |
|-----|----------|--------------|---------------|-------|
| `gpu-test` | 8000 | — | — | Quick testing |
| `gpu-short` | 5000 | 44 | — | Default short GPU |
| `gpu-medium` | 2000 | 20 | 24 | Group limit: 160 GPUs |
| `gpu-long` | 1000 | 16 | 16 | Long-running |

## QOS: `ailab` (partition default)

| QOS | Priority | MaxGPUs/User | Notes |
|-----|----------|--------------|-------|
| `ailab` | 0 | 16 | Often unused for priority after the `gpu-*` rewrite |

## QOS: `pli-*`

| QOS | Priority | MaxGPUs/User | Notes |
|-----|----------|--------------|-------|
| `pli-short` | 5000 | 64 | also max 10 nodes |
| `pli-medium` | 3000 | 32 | max 8 nodes; group 176 GPUs |
| `pli-long` | 0 | 8 | group 8 GPUs |
| `pli-high` | 10000 | 64 | per user and per account (re-verify) |
| `pli-low` | 0 | 16 | group 48 GPUs |
| `pli-cp` | 8000 | 8 | per account |
| `pli-lc` | 0 | 16 | max 16 nodes; group 30 GPUs |
