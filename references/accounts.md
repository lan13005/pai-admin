# ailab / ailab-p accounts and access

Coordinator-facing rules for granting access and capping project usage.
Recorded limits: [values.md](values.md). Hard submit rules: [submit-restrictions.md](submit-restrictions.md).

## Account tree

```text
ailab
╚══ ailab-p
    └── <project subaccounts>
```

`ailab` is the root account and `ailab-p` a child. Child accounts can be created at any level.
Inspect the live tree with:

```bash
slurmtree [-a ACCOUNT] [-c CLUSTER] [-q] [-u]     # -q shows QOS, -u shows users
slurmtree -a ailab -q -u
```

## Faculty sponsor

```bash
finger <user>     # Office: <dept>, <faculty sponsor>
```

Use **Office** to map a netid to their faculty sponsor (and department).

## Two access gates

Both partitions use `AllowGroups=cses,ailab`. **Unix group ≠ Slurm account** — a user needs both.

| Gate | Check | Who controls |
|------|-------|--------------|
| Unix group | `getent group ailab` / `id <user>` | **Sysadmins** — coordinators cannot edit LDAP/groups |
| Slurm account | `slurmtree -a ailab -q -u`, `sacctmgr show assoc where User=<user>` | **Coordinators** on `ailab` / `ailab-p` (via `sacctmgr`) |

In practice:

- New user in neither `ailab` nor `cses` → sysadmins must add the group first.
- User already in `ailab`/`cses` → a coordinator can grant **project** access under `ailab-p`
  without sysadmin involvement.
- Verify project membership with `slurmtree`, not only `getent group ailab`.

## Account / partition rules

From `slurm.conf` (re-check if policy may have changed):

- **`ailab`**: `AllowGroups=cses,ailab`, `DenyAccounts=ailab-p` — refuses `ailab-p` and its children.
- **`ailab-p`**: `AllowGroups=cses,ailab`, `AllowAccounts=ailab-p` — requires `ailab-p` or a child.
- Create project work as **children of `ailab-p`**, never under bare `ailab`.
- Account names are **unique cluster-wide** and must not collide with usernames or groups; prefer
  an `ailab-` prefix.
- Other Della partitions may not block `ailab-p` yet, and further priority / `job_submit` changes
  may be pending. Verify with `sacctmgr` or a test submit.

```bash
sbatch -p ailab-p --account=<ailab-p or child> …
```

## Project allocations (GrpTRESMins)

Relevant accounting knobs (confirm live with `scontrol show config`):

```text
PriorityUsageResetPeriod=None          # usage does not auto-reset
AccountingStorageEnforce=associations,limits,qos
```

`safe` is typically **not** enabled, so a job may start even when the remaining allocation cannot
cover its full request — it can then be killed when the account hits the limit. Once the cap is
reached, later jobs pend with `AssocGrpGRESMins` / `AssocGrpBillingMinutes`
([troubleshooting.md](troubleshooting.md#why-is-my-job-not-running)).

- Cap GPU-hours with **`GrpTRESMins`** on the project account. Units are TRES-minutes: 10 000
  GPU-hours → `gres/gpu=600000`.
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

- H200 shared pool; group adds are sysadmin-owned (see the two gates above).
- Associations are allowed on `ailab` itself, not on `ailab-p` children (`DenyAccounts=ailab-p`).
- Effective priority QOS is usually `gpu-*` after the submit rewrite — see [priority.md](priority.md).
- Submit policy in brief — requires GPUs, ≤ 8 CPUs per GPU per node, rejects the `pli` account.
  Authoritative list: [submit-restrictions.md](submit-restrictions.md).

### ailab-p

- Same H200 nodes as `ailab`; project subaccounts are coordinator-managed.
- Needs Unix `ailab`/`cses` **plus** account `ailab-p` or a child.
- Confirm limits and priority live: `scontrol show partition ailab-p`, `slurmtree -a ailab -q`,
  `sacctmgr show qos`.

### pli

- H100 pool with partition-specific `pli-*` QOS (early return in the Lua filter).
- Submit policy in brief — requires GPUs, time ≤ 3 days, anti-fragmentation on multinode, node
  excludes for single-GPU jobs, no mixing with `ailab*`. Authoritative list:
  [submit-restrictions.md](submit-restrictions.md).
- Audience and user-facing model: [pli-partition.md](pli-partition.md).
