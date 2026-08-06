---
name: pai-admin
description: Slurm administration for Princeton's Della cluster, especially the ailab / ailab-p and pli partition families. Use for GPU, node, or queue status; user and project access; sponsor lookup; QOS and priority; accounts and allocations; submission policy; Slurm configuration and plugins; or cluster diagnostics.
---

# Della Admin Guide

Cluster: **Della** (Princeton Research Computing), scheduler **Slurm**. Its restricted GPU partition
families are **`ailab*`** (H200) and **`pli*`** (H100), with different access and QOS schemes.

**Prefer live reads over recorded values.** Every table in `references/` is a snapshot; re-verify
with the commands below whenever policy may have changed.

## Quick start

```bash
gfree                                  # free GPUs by type/partition/cores/mem/Slurm directives
shownodes -p <partition>               # node states, free cpus/gpus, memory, Slurm Features (os, nvme, intel, gpu group)
squeue -p <partition>                  # queue + pending reasons
scontrol show partition <partition>    # AllowGroups, AllowAccounts, MaxTime, TRESBillingWeights
slurmtree -a <account> -q -u           # account tree with QOS (-q) and users (-u) - Slurm Coordinator
qos                                    # QOS priorities and limits
```

## Workflows

### Diagnose a pending job

1. `scontrol show job <jobid>` — read `Reason`, `Partition`, `QOS`, `Account`. The submit filter
   assigns the QOS, so it is usually **not** the partition default and the routing differs by
   partition family — see [references/priority.md](references/priority.md).
2. `qos` — compare that QOS against the other tiers' priorities and per-user GPU ceilings. This one
   command explains most `Priority` and `QOSMaxGRESPerUser` holds. Check whether the partition has its
   own QOS first (`scontrol show partition <p>` → `QoS=`); if so, **its** limits bind, not the tier's.
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

Run `jobstats <jobid>` to see the request, state, timing, CPU/GPU/memory utilization, and
underutilization flags. Includes notes and warnings that helps you quickly identify problems.
`jobstats` is provided by Job Defense Shield.

### Check or grant a user's access

1. Identity / sponsor: `finger <user>` — read **Office** (`<dept>, <faculty sponsor>`).
2. Read the partition's gates: `scontrol show partition <partition> | tr ' ' '\n' |
   grep -E '^(AllowGroups|AllowAccounts|DenyAccounts|AllowQos|QoS)='`.
3. Unix gate: `id -nG <user> | tr ' ' '\n' | grep -Fx <group>` — **sysadmins** own this.
4. Slurm account + QOS gates:
   `sacctmgr show assoc where User=<user> format=Cluster,Account,User,Partition,QOS%200` and
   `slurmtree -a <account> -q -u` — **coordinators** own these.
5. All three gates must pass. `ailab*` is gated by account, `pli*` mostly by QOS on the subaccount.
   Family names, tree layout, and allocation caps: [references/accounts.md](references/accounts.md).

### Investigate a rejected submission

1. Read the exact `sbatch: error:` text. Rejections come from the **controller**, not the local
   `sbatch` binary.
2. Check the plugin chain: `scontrol show config | egrep -i 'JobSubmitPlugins|PluginDir'`
   (typically `list_accounts,lua` with `PluginDir = /usr/lib64/slurm`).
3. Map the message to a rule in [references/submit-restrictions.md](references/submit-restrictions.md),
   then confirm against `/etc/slurm/job_submit.lua`.

## Reference material

| File | Contents |
|------|----------|
| [references/accounts.md](references/accounts.md) | Access gates, account trees, sponsors, and allocations |
| [references/priority.md](references/priority.md) | Multifactor priority and job QOS vs partition QOS |
| [references/submit-restrictions.md](references/submit-restrictions.md) | `job_submit.lua` rules, QOS routing, and plugin inspection |
| [references/slurm-config.md](references/slurm-config.md) | `/etc/slurm/` file map and how to read the effective config |
| [references/pli-partition.md](references/pli-partition.md) | User-facing PLI model: audience, intent, best practices, storage, support |
| [references/values.md](references/values.md) | Recorded inventory and QOS snapshots for quick citation |
| [references/troubleshooting.md](references/troubleshooting.md) | Pending reasons, node health, and resource use |

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
