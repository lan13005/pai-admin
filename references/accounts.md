# Accounts and access on Della

Slurm Coordinator-facing model for granting access and capping project usage. The same three-layer
framework applies to every restricted partition family, but the active gates and names differ.
Recorded limits: [values.md](values.md). Hard submit rules:
[submit-restrictions.md](submit-restrictions.md).

## The three gates (any partition)

A job must clear all three, and different people own them:

| Gate | Enforced by | Check | Owner |
|------|-------------|-------|-------|
| Unix group | partition `AllowGroups` | `getent group <group>`, `id <user>` | **Sysadmins** — coordinators cannot edit LDAP/groups |
| Slurm account | partition `AllowAccounts` / `DenyAccounts`, plus an association | `sacctmgr show assoc where User=<user>`, `slurmtree -a <account> -q -u` | **Coordinators** on that account |
| Slurm QOS | the QOS list on the association (`AccountingStorageEnforce=...,qos`) | `slurmtree -a <account> -q` | **Coordinators** on that account |

The QOS gate is easy to miss. When `job_submit.lua` forces a specific QOS for a partition, a user
whose association does not list that QOS is rejected even though `AllowGroups=ALL` and
`AllowAccounts=ALL`. That is exactly how the `pli` and `pli-lc` partitions are gated.

**Unix group ≠ Slurm account**, and a name can be both: `cses` is a Unix group in `AllowGroups`
*and* a Slurm account that `job_submit.lua` compares against. Always say which one you mean.

Read a partition's gates live rather than trusting any table here:

```bash
scontrol show partition <partition> | tr ' ' '\n' | grep -E '^(AllowGroups|AllowAccounts|DenyAccounts|AllowQos|QoS)='
```

## Account trees (any partition)

Accounts form a tree: a root account, child accounts, and project subaccounts under those. Children
can be created at any level. Account names are **unique cluster-wide** and must not collide with
usernames.

```bash
slurmtree [-a ACCOUNT] [-c CLUSTER] [-q] [-u]     # -q shows QOS, -u shows users
```

Partitions restrict *which* accounts may submit, so where a project sits in the tree decides which
partitions it can reach. Grant project access by adding the user to a subaccount rather than
widening the root. Verify membership with `slurmtree`, not only `getent group`.

## Faculty sponsor

```bash
finger <user>     # Office: <dept>, <faculty sponsor>
```

Use **Office** to map a netid to their faculty sponsor (and department).

## Project allocations (GrpTRESMins)

Relevant accounting knobs (confirm live with `scontrol show config`):

```text
PriorityUsageResetPeriod=NONE          # usage does not auto-reset
AccountingStorageEnforce=associations,limits,qos
```

`safe` is **not** enabled, so a job may start even when the remaining allocation cannot cover its
full request — it can then be killed when the account hits the limit. Once the cap is reached, later
jobs pend with `AssocGrpGRESMins` / `AssocGrpBillingMinutes`
([troubleshooting.md](troubleshooting.md#why-is-my-job-not-running)).

- Cap cumulative GPU-hours with **`GrpTRESMins`** on the project account. Units are TRES-minutes.
- Cap *concurrent* GPUs with **`GrpTRES`** instead (`gres/gpu=32` means 32 at once, forever).
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

## ailab / ailab-p — gated by account

```text
ailab
└── ailab-p          # project subaccounts go here
```

Same H200 nodes for both partitions; the split is which account may submit.

- Unix gate: `AllowGroups=cses,ailab` on **both** partitions.
- `ailab`: `DenyAccounts=ailab-p` — refuses `ailab-p` and its children, so associations live on
  `ailab` itself.
- `ailab-p`: `AllowAccounts=ailab-p` — requires `ailab-p` or a child.
- Create project work as **children of `ailab-p`**, never under bare `ailab`; prefer an `ailab-`
  prefix.
- New user in neither `ailab` nor `cses` → sysadmins must add the group first. A user already in one
  of those groups can be granted project access by a coordinator alone.
- QOS: the account tree grants `gpu-*`, and the submit filter rewrites walltime into a `gpu-*` tier
  ([priority.md](priority.md)). Each partition also carries a partition QOS (`ailab`, `ailab-p`) whose
  limits **override** the tier's.
- Submit policy in brief — requires GPUs, ≤ 8 CPUs per GPU per node, rejects the `pli` account.
  Authoritative list: [submit-restrictions.md](submit-restrictions.md).

```bash
sbatch -p ailab-p --account=<ailab-p or child> …
scontrol show partition ailab-p
slurmtree -a ailab -q -u
```

## pli* — gated by QOS

```text
pli
├── <project subaccount>          # one per proposal/group
└── …                             # QOS list on the subaccount sets the tier
```

Same tree shape as `ailab`, but the entitlement tier lives in the **QOS list on the subaccount**
rather than in partition `AllowAccounts`:

- `pli` and `pli-lc` are `AllowGroups=ALL` / `AllowAccounts=ALL`. The filter forces `pli-low` and
  `pli-lc` respectively, so the effective gate is whether the association carries that QOS.
- `pli-c` and `pli-p` add a Unix gate: `AllowGroups=cses,pli` (the `pli` group is real and
  populated). `pli-c` also rewrites the account to `pli` unless it is already `cses`.
- Most subaccounts carry only `pli-low`; large-campus projects carry `pli-lc`. Grant a tier by
  adding the QOS to the subaccount, not by moving the account.
- Submit policy in brief — requires GPUs, time ≤ 3 days, anti-fragmentation on multinode, node
  excludes for single-GPU jobs, no mixing with `ailab*`. Authoritative list:
  [submit-restrictions.md](submit-restrictions.md).
- Audience and user-facing model: [pli-partition.md](pli-partition.md).

```bash
slurmtree -a pli -q -u
sacctmgr modify account where name=<project> set qos+=pli-lc   # grant a tier
```
