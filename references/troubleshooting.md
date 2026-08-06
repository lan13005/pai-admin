# Troubleshooting

## Why is my job not running?

A pending job is held by **one of two independent gates**, and the fix for one does nothing for the
other:

| Gate | Question | Set by |
|------|----------|--------|
| **Order** | When does the scheduler get around to *considering* this job? | Multifactor priority — fairshare, age, QOS, job size ([priority.md](priority.md)) |
| **Availability** | When considered, does the request actually *fit* right now? | Free GPUs, node shape and fragmentation, QOS/account limits, node health |

**Priority moves you to the head of the line; it does not create GPUs.** A top-priority 4-node job
still waits until 4 nodes free up together, and raising a QOS or fairshare will not shorten that
wait. Conversely, if the cluster is idle and the job still sits, the hold is a limit or a
dependency, not ordering.

Start by reading the reason and routing on it:

```bash
squeue -j <jobid> -o "%.10i %.9P %.12q %.10a %.8u %.2t %.10M %.6D %.8b %R"
scontrol show job <jobid>              # Reason, Partition, QOS, Account, ReqTRES
```

| `Reason` | Gate | Where to look |
|----------|------|---------------|
| `Priority` | order | [priority.md](priority.md), then check availability anyway (below) |
| `Resources` | availability | `gfree`, `shownodes -p <partition>` |
| `QOSMaxGRESPerUser`, `QOSGrpGPULimit`, `QOSMaxNodePerUser` | availability (limit) | `qos`, [values.md](values.md) |
| `AssocGrpGRESMins`, `AssocGrpBillingMinutes` | availability (allocation) | `GrpTRESMins` on the account — [accounts.md](accounts.md) |
| `InvalidQOS`, `InvalidAccount` (rejected, not pending) | access | the account's QOS list — [accounts.md](accounts.md) |
| `ReqNodeNotAvail`, `Nodes required for job are DOWN/DRAINED` | availability (health) | node drain section below |
| `Dependency`, `JobHeldUser`, `JobHeldAdmin`, `BeginTime` | neither | the job's own state; nothing to do with the cluster |

Reason strings vary by Slurm version and are truncated in default `squeue` output — take them from
`scontrol show job`, not from a screenshot.

Whether you can target a better QOS depends on partition family:

- **`ailab` / `ailab-p`**: walltime selects the `gpu-*` tier. Shortest honest `--time` plus a smaller
  shape (fewer GPUs / nodes) raises QOS priority and packs / backfills more easily.
- **`pli*`**: the partition (not walltime) picks the QOS, except walltime bins inside `pli-c`. A
  different tier means a different partition or a QOS grant on the account
  ([priority.md](priority.md), [accounts.md](accounts.md)).

## "Priority"

Queued behind higher-priority work. Break the score down with `sprio -j <jobid>` and
`sshare -u <user>` — see [priority.md](priority.md). Fairshare often matters more than QOS, and the
submit filter assigns the QOS, so read it off the job rather than the partition.

`Priority` and `Resources` alternate as the cluster fills, so a `Priority` reason does **not** mean
priority is the whole story. Check availability too before promising that a QOS or fairshare change
will help:

```bash
scontrol show job <jobid> | rg -o 'StartTime=\S+'   # backfill's own estimate, if any
gfree
```

## "Resources"

The request does not fit the free pool right now. Free GPUs in aggregate are not enough — the job
needs them in the right **shape** (per node, and on nodes the job is eligible for):

```bash
gfree                                                  # free GPUs by type/partition
shownodes -p <partition>                               # per-node state and free GPUs
squeue -p <partition> -t R -o "%.10i %.8u %.6D %.8b %R"  # who holds the nodes
```

Eight free GPUs spread one-per-node will not start a `-N 1 --gres=gpu:8` job. Either wait, or have
the user relax the shape (fewer GPUs per node, `-N` unset, shorter `--time` so backfill can slot
them in).

## "QOSMaxGRESPerUser"

The user hit a per-user GPU ceiling. Find which QOS actually sets it before advising anything — where
a partition has its own QOS, that one overrides the job's QOS limits
([priority.md](priority.md#how-the-two-qos-interact)):

```bash
scontrol show job <jobid> | rg -o 'QOS=\S+'                 # job QOS
scontrol show partition <p> | tr ' ' '\n' | grep '^QoS='    # partition QOS, if any
squeue -u <user> -t R -h -O "tres-alloc:100"                # what they already hold
```

Whether a shorter `--time` raises the ceiling:

- **`ailab` / `ailab-p`**: no. The partition QOS pins the ceiling at 16 (`ailab`) or 24 (`ailab-p`)
  for every `gpu-*` tier. A shorter `--time` buys queue position, not headroom.
- **general GPU partitions**: yes. With no partition QOS, the `gpu-*` tier's own ceiling applies
  (`gpu-short` 44, `gpu-medium` 20, `gpu-long` 16).
- **`pli*`**: no. The partition fixes the QOS, so the user needs a different partition or a QOS grant
  on their account ([accounts.md](accounts.md)).

Recorded ceilings: [values.md](values.md).

## Rejected at submit: "You have to specify an account" / "Invalid account or account/partition combination"

Not a pending reason — the controller refused the job. The allowed-account message comes from
**`job_submit_list_accounts.so`** (`list_accounts` in `JobSubmitPlugins`), not from
`job_submit.lua`. A second line is usually Slurm's stock association error. Confirm associations,
then see [submit-restrictions.md](submit-restrictions.md).

An invalid-QOS rejection on `pli` / `pli-lc` usually means the account was never granted the QOS the
filter forces for that partition — that is the access gate there, not a typo
([accounts.md](accounts.md)).

```bash
sacctmgr show assoc where User=<user> format=Cluster,Account,User,Partition,QOS%200
```

## Node in "drain" or "down" state

```bash
scontrol show node <node> | grep -i reason
```

## Who is using resources

```bash
squeue -p <partition> -t R -o "%.10i %.9P %.20j %.8u %.2t %.10M %.6D %.4C %.8b %R"
```
