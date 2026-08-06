---
name: pai-admin
description: Slurm administration for the Della HPC cluster (ailab, ailab-p, pli). Use when the user asks about GPU availability, node status, job queues, user access, faculty sponsor / office lookup (finger), adding users to ailab/ailab-p, coordinator account management, QOS limits, multifactor priority / sprio, partition info, account trees (slurmtree), ailab-p subaccounts, cluster diagnostics, job_submit.lua / Slurm config paths, or account/submit plugins.
---

# Della Admin Guide

Cluster: **Della** (Princeton Research Computing), scheduler **Slurm**.
Restricted GPU partitions: **`ailab`** / **`ailab-p`** (H200), **`pli*`** (H100).

**Prefer live reads over recorded values.** Every table in `references/` is a snapshot; re-verify
with the commands below whenever policy may have changed.

## Quick start

```bash
gfree                                  # free GPUs by type/partition/cores/mem/Slurm directives
shownodes -p ailab                     # node states, free cpus/gpus, memory, Slurm Features (os, nvme, intel, gpu group)
squeue -p ailab                        # queue + pending reasons
scontrol show partition ailab          # AllowGroups, MaxTime, TRESBillingWeights
slurmtree -a ailab -q -u               # account tree with QOS (-q) and users (-u) - Slurm Coordinator
qos                                    # QOS priorities and limits
```

## Workflows

### Diagnose a pending job

1. `scontrol show job <jobid>` — read `Reason`, `Partition`, `QOS`, `Account`. The QOS was assigned
   from walltime at submit time, so it is usually **not** the partition default.
2. `qos` — compare that QOS against the other tiers' priorities and per-user GPU ceilings. This one
   command explains most `Priority` and `QOSMaxGRESPerUser` holds.
3. Route on the reason with the table in
   [references/troubleshooting.md](references/troubleshooting.md#why-is-my-job-not-running). Every
   hold is either **order** (priority) or **availability** (fit, limits, node health), and the fix
   for one does nothing for the other.
4. For `Priority`, break the score down with `sprio -j <jobid>` and `sshare -u <user>`; the formula
   and the `pli-*` vs `gpu-*` routing are in [references/priority.md](references/priority.md), the
   walltime bins themselves in [references/submit-restrictions.md](references/submit-restrictions.md).
   Priority sets queue order, not packing — check `gfree` and `shownodes -p <partition>` before
   blaming the score.

### Check utilization of a running job

1. `jobstats` is part of the Job Defense Shield system, see [Reference Material](#reference-material) below.
2. Running `jobstats <jobid>` will show you important information about the job including:
   - user, job state, cpu/gpu/mem request, qos/partition, timing, and more.
3. More importantly, it shows (cpu/gpu/mem) utilization averages and flags poor utilization!

### Check or grant a user's access

1. Identity / sponsor: `finger <user>` — read **Office** (`<dept>, <faculty sponsor>`).
2. Unix gate: `getent group ailab | grep <user>` and `id <user>` — **sysadmins** own this.
3. Slurm gate: `sacctmgr show assoc where User=<user> format=Cluster,Account,User,Partition,QOS%200`
   and `slurmtree -a ailab -q -u` — **coordinators** own this.
4. Both gates must pass. Full rules, subaccount layout, and allocation caps:
   [references/accounts.md](references/accounts.md).

### Investigate a rejected submission

1. Read the exact `sbatch: error:` text. Rejections come from the **controller**, not the local
   `sbatch` binary.
2. Check the plugin chain: `scontrol show config | egrep -i 'JobSubmitPlugins|PluginDir'`
   (typically `list_accounts,lua` with `PluginDir = /usr/lib64/slurm`).
3. Map the message to a rule in [references/submit-restrictions.md](references/submit-restrictions.md),
   then confirm against `/etc/slurm/job_submit.lua`.

### Answer a PLI user question

Which partition to use, why an empty queue still waits, CPU/mem per GPU, single-GPU and array-job
guidance, storage, and support contacts: [references/pli-partition.md](references/pli-partition.md).

## Reference material

| File | Contents |
|------|----------|
| [references/accounts.md](references/accounts.md) | `ailab` / `ailab-p` account tree, faculty sponsor via `finger`, the two access gates, partition account rules, `GrpTRESMins` allocations |
| [references/priority.md](references/priority.md) | Multifactor priority formula, where each factor comes from, submit-time QOS routing (`gpu-*` vs `pli-*`) |
| [references/submit-restrictions.md](references/submit-restrictions.md) | Submit-time rules from `job_submit.lua`, the walltime → `gpu-*` bins, RPC → plugin chain, inspecting compiled `.so` plugins |
| [references/slurm-config.md](references/slurm-config.md) | `/etc/slurm/` file map and how to read the effective config |
| [references/pli-partition.md](references/pli-partition.md) | User-facing PLI model: audience, intent, best practices, storage, support |
| [references/values.md](references/values.md) | Recorded inventory and QOS snapshots for quick citation |
| [references/troubleshooting.md](references/troubleshooting.md) | Order vs availability model, `Reason` → where-to-look routing table, node drain/down, who-is-using-what |

Additional references:

- Princeton RC uses [Job Defense Shield](https://princetonuniversity.github.io/job_defense_shield/) to identify and
reduce instances of underutilization of the cluster. This is the system that sends automated email alerts, automatically
cancels GPU jobs at 0% utilization, and create reports.
- [Removing Tedium](https://github.com/PrincetonUniversity/removing_tedium) is collection of docs that can help improve your
productivity. Are you tired of Duo? Do you waste time entering your password every time you log in or do a file transfer? 
Do you want to automate repetitive tasks? Click that button.

## Useful commands

```bash
finger <user>                          # Office = dept, faculty sponsor, map netid to office
gpudash                                # Show GPU utilization across all nodes in the cluster, which nodes busy
scontrol show job <jobid>
scontrol show node <nodename>
sacct -j <jobid> --format=JobID,JobName,Partition,State,Elapsed,MaxRSS,MaxVMSize,AllocTRES%40
seff <jobid>
squeue -u <user>
sshare -u <user>
sprio -j <jobid>                       # weighted points;  -n for normalized factors
sacctmgr show qos format=Name,Priority,GrpTRES,MaxTRESPU,MaxTRESPA,MaxJobsPU
scancel <jobid>
```
