# Job priority on Della

Della uses **`PriorityType = priority/multifactor`**. Prefer live reads over memorized weights.

## Partition vs QOS

| Concept | Controls |
|---------|----------|
| **Partition** | Which **nodes** are eligible |
| **QOS** | Queue **priority** contribution plus **limits** (max GPU/user, group caps, etc.) |

Multiple partitions can share the same nodes with different defaults and limits.

## Formula

```text
Priority = PriorityWeightAge       × age_factor
         + PriorityWeightFairShare × fairshare_factor
         + PriorityWeightJobSize   × jobsize_factor
         + PriorityWeightPartition × partition_factor
         + PriorityWeightQOS       × qos_factor
         + PriorityWeightTRES      × tres_factor
```

Factors are normalized (typically 0–1; age grows with wait time, capped by `PriorityMaxAge`).
`sprio` prints weighted point contributions, `sprio -n` prints normalized factors.

## Where each piece comes from

| Piece | Source |
|-------|--------|
| Weights / flags / max age | `scontrol show config` → `PriorityWeight*`, `PriorityFlags`, `PriorityMaxAge`, `PriorityDecayHalfLife` |
| Job's QOS name | `scontrol show job <jobid>` or `squeue -o "%i %q"` |
| QOS config priority + limits | `sacctmgr show qos …` or `qos` |
| Point breakdown for one job | `sprio -j <jobid>` |
| Fairshare | `sshare -u <user>` (also appears in `sprio`) |

## QOS contribution

QOS priority is normalized against the **highest QOS priority on the whole cluster**, not per
partition:

```text
qos_factor = qos_priority / max_qos_priority_cluster
QOS_points = PriorityWeightQOS × qos_factor
```

**Rule of thumb:** when `PriorityWeightQOS=8000` and max cluster QOS priority is `20000`,
`QOS_points ≈ qos_priority × 0.4` (e.g. `gpu-short` 5000 → 2000 pts). Re-derive if either
number drifts.

### Scale factors (snapshot — re-check)

```bash
scontrol show config | grep Priority
```

| Parameter | Recorded value |
|-----------|----------------|
| `PriorityWeightFairShare` | 12000 |
| `PriorityWeightAge` | 10000 |
| `PriorityWeightJobSize` | 10000 |
| `PriorityWeightQOS` | 8000 |
| `PriorityWeightPartition` | 1 |
| `PriorityWeightAssoc` | 0 |
| `PriorityFlags` | `MAX_TRES,NO_NORMAL_PART` |
| `PriorityMaxAge` | 30 days |
| `PriorityDecayHalfLife` | 15 days |

```bash
sacctmgr -n -P show qos format=Name,Priority,MaxTRESPerUser,GrpTRES
sacctmgr -n -P show qos format=Name,Priority | awk -F'|' '{print $2, $1}' | sort -rn | head -5
```

**Typical ordering influence on ailab:** fairshare often dominates QOS, age can overtake QOS for
long-pending jobs, and partition weight is usually negligible. Confirm with `sprio`, don't assume.

## Submit-time QOS routing

Users rarely pick their own QOS: `/etc/slurm/job_submit.lua` assigns one at submit time. **The QOS a
job competes under is usually not the partition default**, so read the assigned QOS off the job
rather than the partition.

There are two routes, and which one a job takes depends only on its partition:

- **`pli*`** — `set_plic_qos()` / explicit assignment picks a `pli-*` QOS, then the block **returns
  early**. These jobs never reach the `gpu-*` rewrite.
- **everything else** (including `ailab` / `ailab-p`) — `getQOS(time_limit)` assigns a walltime tier,
  and GPU jobs then get that tier rewritten into a `gpu-*` name.

### The `gpu-*` split

`getQOS()` bins on `--time` in minutes, and GPU jobs then map `vlong → long` and prefix `gpu-`. The
bin table, the off-by-a-minute cutoffs, and the `gputest` partition redirect are in
[submit-restrictions.md](submit-restrictions.md#walltime-bins-general-gpu-flow).

What matters for priority: **`--time` picks the tier, and the tier carries both the QOS priority and
the per-user GPU ceiling.** A user asking for more walltime than they need is silently trading away
priority and GPU headroom. Recorded priorities per tier: [values.md](values.md).

### Reading the routing live

```bash
qos                                                      # QOS priorities and limits, all tiers
scontrol show job <jobid> | rg -o 'QOS=\S+'              # what one job actually got
squeue -p ailab -h -o "%q" | sort | uniq -c | sort -rn   # effective QOS mix in a partition
rg -n 'getQOS|_MINS|gpu-|pli-' /etc/slurm/job_submit.lua # the routing source itself
```

Use `qos` first when someone asks "why is my job behind?" — it shows the priority and per-user GPU
ceiling of every tier side by side, which is usually enough to explain both the ordering and a
`QOSMaxGRESPerUser` hold.

Priority affects **queue order, not packing**. A large multi-node GPU job can still wait on
fragmentation even with high QOS points — the order-vs-availability split and the reason-code
routing table live in [troubleshooting.md](troubleshooting.md#why-is-my-job-not-running).
