# Job priority on Della

Della uses **`PriorityType = priority/multifactor`**. Prefer live reads over memorized weights.

## The three layers

A job is shaped by three things, and conflating them causes most wrong answers:

| Layer | Sets | Read with |
|-------|------|-----------|
| **Partition** | Which **nodes** are eligible | `scontrol show partition <p>` |
| **Job QOS** (`gpu-*`, `pli-*`) | The **priority** contribution, and limits where no partition QOS overrides them | `scontrol show job <jobid>` → `QOS=` |
| **Partition QOS** (`scontrol show partition <p>` → `QoS=`) | **Limits only**, and it *wins* over the job QOS | `sacctmgr show qos <name>` |

Multiple partitions can share the same nodes with different defaults and limits.

### How the two QOS interact

Split them by what they control:

- **Priority comes from the job QOS.** The partition QOS priority is not used. A `gpu-short` job on
  `ailab` scores `5000/20000 × 8000 = 2000` QOS points even though the partition QOS `ailab` has
  priority 0 — verify with `sprio -j <jobid>`. See [Priority Formula](#priority-formula) for more formula.
- **Limits come from the partition QOS where it defines one.** No QOS on this cluster carries the
  `OverPartQOS` flag, so where the partition QOS sets a limit, it replaces the job QOS value.

`ailab`, `ailab-p`, and `pli-c` each have a partition QOS; `pli`, `pli-lc`, and `pli-p` do not.

**Worked example — the per-user GPU ceiling on `ailab` is 16 for every tier.** `gpu-short` allows 44
GPUs per user and `gpu-medium` allows 20, but the partition QOS `ailab` sets 16, so that is the
binding number. Users in both tiers pend with `QOSMaxGRESPerUser` at 16. On `ailab-p` the partition
QOS raises it to 24 (group 48). On general GPU partitions there is no partition QOS, so the `gpu-*`
ceilings do apply. Recorded numbers: [values.md](values.md).

```bash
scontrol show partition <p> | tr ' ' '\n' | grep '^QoS='   # is there a partition QOS?
sacctmgr show qos where name=<partition-qos>,<job-qos> format=Name,Priority,GrpTRES,MaxTRESPU,Flags
```

## Priority formula

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

| Piece | Source |
|-------|--------|
| Weights / flags / max age | `scontrol show config` → `PriorityWeight*`, `PriorityFlags`, `PriorityMaxAge`, `PriorityDecayHalfLife` |
| Job's QOS name | `scontrol show job <jobid>` or `squeue -o "%i %q"` |
| QOS config priority + limits | `sacctmgr show qos …` or `qos` |
| Point breakdown for one job | `sprio -j <jobid>` |
| Fairshare | `sshare -u <user>` (also appears in `sprio`) |

### QOS contribution

QOS priority is normalized against the **highest QOS priority on the whole cluster**, not per
partition:

```text
qos_factor = qos_priority / max_qos_priority_cluster
QOS_points = PriorityWeightQOS × qos_factor
```

With `PriorityWeightQOS=8000` and a cluster max of `20000`, `QOS_points ≈ qos_priority × 0.4`
(`gpu-short` 5000 → 2000 pts, `gpu-long` 1000 → 400 pts). Re-derive if either number drifts:

```bash
sacctmgr -n -P show qos format=Name,Priority | awk -F'|' '{print $2, $1}' | sort -rn | head -5
```

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
| `PriorityWeightTRES` | `CPU=1,Mem=1,GRES/gpu=1` |
| `PriorityFlags` | `MAX_TRES,NO_NORMAL_PART` |
| `PriorityMaxAge` | 30 days |
| `PriorityDecayHalfLife` | 15 days |

**Typical ordering influence on ailab:** fairshare often dominates QOS, age can overtake QOS for
long-pending jobs, and partition weight is usually negligible. Confirm with `sprio`, don't assume.

## Submit-time QOS routing

Users rarely pick their own QOS: `/etc/slurm/job_submit.lua` assigns one at submit time. **The QOS a
job competes under is usually not the partition default**, so read the assigned QOS off the job
rather than the partition. Exact mappings and walltime bins are in
[submit-restrictions.md](submit-restrictions.md).

There are two routes, and which one a job takes depends only on its partition.

### `pli*` route — partition picks the QOS

The PLI block chooses a `pli-*` QOS and then **returns early**, so these jobs never reach the
`gpu-*` rewrite, unlike `ailab` jobs. The partition chooses the QOS except within `pli-c`, where
walltime selects a tier.

Because the partition fixes the QOS, the user must have been **granted** that QOS on their account —
that is the access gate on `pli` and `pli-lc`, see [accounts.md](accounts.md).

### `gpu-*` route — walltime picks the tier

Everything else, including `ailab` / `ailab-p`, uses walltime to select a tier; GPU jobs then receive
the corresponding `gpu-*` QOS.

What matters for priority: **`--time` picks the tier, and the tier carries the QOS priority.** A user
asking for more walltime than they need is silently trading away queue position. Whether it also
costs GPU headroom depends on the partition QOS — on `ailab` it does not, because the ceiling is 16
either way (see the worked example above).

### Reading the routing live

```bash
qos                                                      # QOS priorities and limits, all tiers
scontrol show job <jobid> | rg -o 'QOS=\S+'              # what one job actually got
squeue -p <partition> -h -o "%q" | sort | uniq -c | sort -rn   # effective QOS mix (try ailab, then pli, pli-c)
rg -n 'getQOS|_MINS|gpu-|pli-' /etc/slurm/job_submit.lua # the routing source itself
```

Use `qos` first when someone asks "why is my job behind?" — it shows the priority and per-user GPU
ceiling of every tier side by side. Then check the partition QOS before quoting a ceiling from it.

Priority affects **queue order, not packing**. A large multi-node GPU job can still wait on
fragmentation even with high QOS points — the order-vs-availability split and the reason-code
routing table live in [troubleshooting.md](troubleshooting.md#why-is-my-job-not-running).
