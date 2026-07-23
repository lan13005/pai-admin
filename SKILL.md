---
name: pai-admin
description: Slurm administration for the Della HPC cluster (ailab, ailab-p, pli). Use when the user asks about GPU availability, node status, job queues, user access, adding users to ailab/ailab-p, coordinator account management, QOS limits, multifactor priority / sprio, partition info, account trees (slurmtree), ailab-p subaccounts, cluster diagnostics, job_submit.lua / Slurm config paths, or account/submit plugins.
---

# Della Admin Guide

## Environment

- **Cluster**: Della (Princeton Research Computing)
- **Scheduler**: Slurm
- **Key partitions**: `ailab` (H200), `ailab-p` (project accounts on ailab hardware), `pli` (H100)

## Skill reference material

- **`01-job-submit-restrictions.md`** (same directory): submission-time rules, `JobSubmitPlugins`, RPC → plugin chain, and relation to `/etc/slurm/job_submit.lua`.
- **`02-pli-partition.md`** (same directory): user-facing PLI model — who uses `pli` / `pli-c` / `pli-lc` / `pli-p`, shared-GPU queues, best practices, single-GPU guidance, storage, support. Consult when questions are about audience, intent, or hygiene rather than live Slurm config.
- **`scripts/`**: UV-runnable Python helpers (`uv run scripts/<name>.py`). Dependencies are declared in a PEP 723 metadata block at the top of each script (assume `uv` is installed):

```python
# /// script
# requires-python = ">=3.12"
# dependencies = [
#     "requests>=2.31.0",
#     "rich",
# ]
# ///
```

## Slurm configuration (paths)

Operator-facing files are under **`/etc/slurm/`**:

| File / area | Role |
|-------------|------|
| **`slurm.conf`** | Partitions, nodes, `AccountingStorage*`, `JobSubmitPlugins`, timeouts, priority weights |
| **`slurmdbd.conf`** | Accounting DB bridge (usually on the DB host) |
| **`gres.conf`** | GRES (e.g. GPU) definitions |
| **`cgroup.conf`** | Cgroup plugin settings |
| **`topology.conf`** | Switch topology (if used) |
| **`plugstack.conf`** | SPANK plugin stack |
| **`job_submit.lua`** | Lua submit filter (when `JobSubmitPlugins` includes `lua`) |

On a login node:

```bash
scontrol show config | head -30
scontrol show config | egrep -i 'JobSubmitPlugins|PluginDir'
```

Typical pattern: `JobSubmitPlugins = list_accounts,lua`, `PluginDir = /usr/lib64/slurm`.

Submit errors (`sbatch: error: …`) come from the **controller** plugin/association path, not the local `sbatch` binary. Full RPC → plugin walkthrough, `.so` string inspection, and restriction tables: **`01-job-submit-restrictions.md`**.

## Quick Diagnostics

### 1. GPU Availability
```bash
gfree
```
GPU types and partition mapping: see **Reference values**.

### 2. Node Status
```bash
shownodes -p <partition>
```

### 3. Job Queue
```bash
squeue -p <partition>
```
Common pending reasons: `Priority`, `Resources`, `QOSMaxGRESPerUser`, `JobArrayTaskLimit`, `Dependency`.

### 4. Partition Details
```bash
scontrol show partition <partition>
```
Key fields: AllowGroups, MaxTime, TotalCPUs, TotalNodes, TRES, TRESBillingWeights.

### 5. User Access
For ailab/ailab-p, both Unix group **and** Slurm account matter (see **Who can grant access**).
```bash
id <user>
getent group ailab | grep <user>          # sysadmin-managed gate
sacctmgr show assoc where User=<user> format=Cluster,Account,User,Partition,QOS%200
slurmtree -a ailab -q -u                   # coordinator-managed tree
```

### 6. Account tree
```bash
slurmtree [-a ACCOUNT] [-c CLUSTER] [-q] [-u]
# e.g. slurmtree -a ailab -q -u
```
Shows the Slurm account hierarchy from an arbitrary root (default cluster: della). `-q` adds QOS; `-u` adds users.

### 7. QOS Overview
```bash
qos
sacctmgr show qos format=Name,Priority,GrpTRES,MaxTRESPU,MaxTRESPA,MaxJobsPU
```

## Job priority (how to calculate)

Della uses **`PriorityType = priority/multifactor`**. Prefer live reads over memorized weights.

### Partition vs QOS

| Concept | Controls |
|---------|----------|
| **Partition** | Which **nodes** are eligible |
| **QOS** | Queue **priority** contribution + **limits** (max GPU/user, group caps, etc.) |

Multiple partitions can share the same nodes with different defaults/limits.

### Formula

```text
Priority = PriorityWeightAge       × age_factor
         + PriorityWeightFairShare × fairshare_factor
         + PriorityWeightJobSize    × jobsize_factor
         + PriorityWeightPartition  × partition_factor
         + PriorityWeightQOS        × qos_factor
         + PriorityWeightTRES       × tres_factor
```

Factors are normalized (typically 0–1; age grows with wait, capped by `PriorityMaxAge`).  
`sprio` prints weighted point contributions; `sprio -n` prints normalized factors.

### Where each piece comes from

| Piece | Source |
|-------|--------|
| Weights / flags / max age | `scontrol show config` → `PriorityWeight*`, `PriorityFlags`, `PriorityMaxAge`, `PriorityDecayHalfLife` |
| Job’s QOS name | `scontrol show job <jobid>` or `squeue -o "%i %q"` |
| QOS config priority + limits | `sacctmgr show qos …` or `qos` |
| Point breakdown for one job | `sprio -j <jobid>` |
| Fairshare | `sshare -u <user>` (also appears in `sprio`) |

### QOS contribution

QOS priority is normalized against the **highest QOS priority on the whole cluster** (not per partition):

```text
qos_factor = qos_priority / max_qos_priority_cluster
QOS_points = PriorityWeightQOS × qos_factor
```

```bash
# weights
scontrol show config | rg -i 'PriorityWeight|PriorityType|PriorityFlags|PriorityMax'

# qos priorities / limits (live)
sacctmgr -n -P show qos format=Name,Priority,MaxTRESPerUser,GrpTRES

# max qos priority (normalization denominator)
sacctmgr -n -P show qos format=Name,Priority | awk -F'|' '{print $2, $1}' | sort -rn | head -5
```

**Typical ordering influence on ailab:** fairshare often dominates QOS; age can overtake QOS for long-pending jobs; partition weight is usually negligible. Confirm with `sprio`, do not assume.

### Submit-time QOS routing (ailab vs PLI)

Behavior is in `/etc/slurm/job_submit.lua` (summary: **`01-job-submit-restrictions.md`**).

- **`ailab`**: GPU jobs usually get a base QoS from walltime (`test|short|medium|vlong`), then rewritten to **`gpu-*`**. The partition-default `ailab` QOS is often **not** what jobs compete under for priority.
- **`pli*`**: sets an explicit `pli-*` QOS and **returns early**, so jobs never hit the `gpu-*` rewrite.

Walltime → base tier cutoffs live in `job_submit.lua` (`TEST_MINS`, `SHORT_MINS`, `MEDIUM_MINS`, `VLONG_MINS`). See **Reference values** for the usual mapping; re-check the Lua file if policy may have changed.

```bash
squeue -p ailab -h -o "%q" | sort | uniq -c | sort -rn   # effective QOS mix
rg -n 'ailab|gpu-|getQOS|pli-' /etc/slurm/job_submit.lua
```

Priority affects **queue order**, not packing. A large multi-node GPU job can still wait on fragmentation even with high QOS points.

## ailab / ailab-p accounts

Live tree (inspect with `slurmtree -a ailab -q -u`):

```text
ailab
╚══ ailab-p
    └── <project subaccounts>
```

### Who can grant access (two gates)

Both partitions use `AllowGroups=cses,ailab`. Unix group ≠ Slurm account.

| Gate | Check | Who controls |
|------|--------|--------------|
| Unix group | `getent group ailab` / `id <user>` | **Sysadmins** — coordinators cannot edit LDAP/groups |
| Slurm account | `slurmtree -a ailab -q -u`, `sacctmgr show assoc where User=<user>` | **Coordinators** on `ailab` / `ailab-p` (via `sacctmgr`) |

Practical:

- New user with no `ailab`/`cses` group → still need sysadmins for the group first.
- User already in `ailab`/`cses` → coordinator can grant **project** access under `ailab-p` without sysadmins.
- Verify project users with `slurmtree`, not only `getent group ailab`.

### Account / partition rules

From `slurm.conf` (re-check if policy may have changed):

- **`ailab`**: `AllowGroups=cses,ailab`, `DenyAccounts=ailab-p` — refuses `ailab-p` and its children.
- **`ailab-p`**: `AllowGroups=cses,ailab`, `AllowAccounts=ailab-p` — requires `ailab-p` or a child.
- Create project work as **children of `ailab-p`**, not under bare `ailab`.
- Account names are **unique cluster-wide** (no overlap with usernames/groups). Prefer an `ailab-` (or similar) prefix.
- Other Della partitions may not yet block `ailab-p`; priority / more `job_submit` changes may still be pending—verify with `sacctmgr` / a test submit.

```bash
sbatch -p ailab-p --account=<ailab-p or child> …
```

### Project allocations (GrpTRESMins)

Relevant accounting knobs (confirm live with `scontrol show config`):

```text
PriorityUsageResetPeriod=None          # usage does not auto-reset
AccountingStorageEnforce=associations,limits,qos
```

`safe` is typically **not** enabled: a job may start even if remaining allocation cannot cover its full request; it can be killed when the account hits the limit.

- Cap GPU-hours with **`GrpTRESMins`** on the project account (units are TRES-minutes; e.g. 10 000 GPU-hours → `gres/gpu=600000`).
- Usage is cumulative until you grant more, reset raw usage, or retire the project.
- Prefer soft admin actions over deleting associations:

| Action | Effect |
|--------|--------|
| Increase `GrpTRESMins` | Grant more GPU-hours |
| `GrpJobs=0` | Pause running capacity (preferred pause) |
| `GrpSubmitJobs=0` | Block new submissions |
| Delete associations | Removes access (destructive; avoid) |

```bash
sacctmgr modify account where name=<project> set GrpTRESMins=gres/gpu=<minutes>
sacctmgr modify account <project> set GrpJobs=0          # pause
sacctmgr modify account <project> set GrpSubmitJobs=0    # block submits
```

## Partition notes

### ailab
- H200 shared pool; `AllowGroups=cses,ailab` (sysadmin group adds — see **Who can grant access**).
- Inventory and limits: **Reference values**.
- Accounts: associations allowed on `ailab`—not `ailab-p` children (`DenyAccounts=ailab-p`).
- Effective priority QOS is usually `gpu-*` after submit rewrite (see **Job priority**).
- Submit policy highlights (detail in **`01-job-submit-restrictions.md`**): requires GPUs; ≤ 8 CPUs/GPU/node; rejects `pli` account.

### ailab-p
- Same H200 nodes as ailab; project subaccounts under `ailab-p` (coordinator-managed).
- Still needs Unix `ailab`/`cses` **plus** account `ailab-p` or a child.
- Confirm limits/priority live: `scontrol show partition ailab-p`, `slurmtree -a ailab -q`, `sacctmgr show qos`.

### pli
- H100 pool with partition-specific `pli-*` QOS (early return in Lua).
- Inventory and QOS rows: **Reference values**.
- Who-uses-which partition, shared-pool model, storage, and user best practices: **`02-pli-partition.md`**.
- Submit policy highlights (detail in **`01-job-submit-restrictions.md`**): requires GPUs; time ≤ 3 days; anti-fragmentation on multinode; single-GPU jobs get node excludes; no mixing with `ailab*`.

## Reference values

Hardcoded inventory and QOS snapshots for quick citation. **Re-verify with the commands above if policy may have changed** (`gfree`, `scontrol show partition`, `sacctmgr show qos`, `job_submit.lua`).

### GPU types

| GPU Model | Mem | Cores/GPU | CPU-Mem/GPU | Partition |
|-----------|-----|-----------|-------------|-----------|
| H200 | 141 GB | 8 | 185 GB | ailab / ailab-p (restricted) |
| H100 | 80 GB | 12 | 120 GB | pli (restricted) |
| A100 80GB | 80 GB | 12 | 240 GB | general (`--constraint=gpu80`) |
| A100 40GB | 40 GB | 64 | 360 GB | general (`--constraint="nomig&gpu40"`) |
| MIG A100 40GB | 40 GB | 6 | 120 GB | general (`--constraint="intel&gpu40"`) |
| MIG A100 10GB | 10 GB | 1 | 32 GB | general (`--constraint=mig`) |

### Partition inventory

| Partition | Nodes | GPUs | Notes |
|-----------|-------|------|-------|
| `ailab` | 18 (`della-i19g[1-3]` … `della-i24g[1-3]`) | 144 × H200 (8/node) | ~1.5 TB/node, 64 CPU/node, MaxTime 15 days; billing CPU=1.0, Mem=0.1G, GPU=12 |
| `ailab-p` | same node family as `ailab` | same H200 pool | project-account partition; confirm live |
| `pli` | — | 336 × H100 | QOS tiers below |

### Walltime → `gpu-*` (ailab / general GPU flow)

From `getQOS()` / rewrite in `job_submit.lua` (constants `TEST_MINS`, `SHORT_MINS`, `MEDIUM_MINS`, `VLONG_MINS`):

| `--time` | Base | Rewritten QOS |
|----------|------|---------------|
| ≤ 1 h | `test` | `gpu-test` |
| ≤ 24 h | `short` | `gpu-short` |
| ≤ 72 h | `medium` | `gpu-medium` |
| > 72 h | `vlong` → `long` | `gpu-long` |

### QOS: `gpu-*`

| QOS | Priority | MaxGPUs/User | MaxNodes/User | Notes |
|-----|----------|--------------|---------------|-------|
| `gpu-test` | 8000 | — | — | Quick testing |
| `gpu-short` | 5000 | 44 | — | Default short GPU |
| `gpu-medium` | 2000 | 20 | 24 | Group limit: 160 GPUs |
| `gpu-long` | 1000 | 16 | 16 | Long-running |

### QOS: `ailab` (partition default)

| QOS | Priority | MaxGPUs/User | Notes |
|-----|----------|--------------|-------|
| `ailab` | 0 | 16 | Often unused for priority after `gpu-*` rewrite |

### QOS: `pli-*`

| QOS | Priority | MaxGPUs/User | Notes |
|-----|----------|--------------|-------|
| `pli-short` | 5000 | 64 | also max 10 nodes |
| `pli-medium` | 3000 | 32 | max 8 nodes; group 176 GPUs |
| `pli-long` | 0 | 8 | group 8 GPUs |
| `pli-high` | 10000 | 64 | per user and per account (re-verify) |
| `pli-low` | 0 | 16 | group 48 GPUs |
| `pli-cp` | 8000 | 8 | per account |
| `pli-lc` | 0 | 16 | max 16 nodes; group 30 GPUs |

## Useful Slurm Commands

```bash
scontrol show job <jobid>
scancel <jobid>
sacct -j <jobid> --format=JobID,JobName,Partition,State,Elapsed,MaxRSS,MaxVMSize,AllocTRES%40
seff <jobid>
squeue -u <user>
scontrol show node <nodename>
sshare -u <user>
sprio -j <jobid>
sprio -n -j <jobid>
sacctmgr show qos format=Name,Priority,GrpTRES,MaxTRESPU,MaxTRESPA,MaxJobsPU
slurmtree -a ailab -q -u
```

## Troubleshooting

### "You have to specify an account" / "Invalid account or account/partition combination"

Allowed-account messaging comes from **`job_submit_list_accounts.so`** (`list_accounts` in `JobSubmitPlugins`), not from `job_submit.lua`. Confirm associations with `sacctmgr show assoc where User=<user> …`. A second line is often Slurm’s stock association error. See **`01-job-submit-restrictions.md`**.

### "QOSMaxGRESPerUser"
User hit the GPU limit for their QOS. Compare `squeue -u <user>` against live QOS limits (`sacctmgr` / **Reference values**).

### "Priority"
Queued behind higher-priority work. Inspect with `sprio -j <jobid>` and `sshare -u <user>` (see **Job priority**). Fairshare often matters more than QOS. Priority does not fix node fragmentation—check `shownodes` / free GPUs per node.

### "Resources"
Not enough free resources. Check `gfree` and `shownodes -p <partition>`.

### Node in "drain" or "down" state
```bash
scontrol show node <node> | grep -i reason
```

### Check who is using resources
```bash
squeue -p <partition> -t R -o "%.10i %.9P %.20j %.8u %.2t %.10M %.6D %.4C %.8b %R"
```
