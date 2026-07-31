# Slurm submission restrictions (Della / PLI / ailab)

This document summarizes **submission-time** restrictions enforced on the Della Slurm cluster, with emphasis on **PLI** and **ailab** GPU partitions.

Recorded QOS and walltime tables: [values.md](values.md). Priority routing: [priority.md](priority.md).

## Where the restrictions live (and how they’re enabled)

- **Lua plugin file**: `/etc/slurm/job_submit.lua`
- **Slurm mechanism**: `JobSubmitPlugins=... ,lua` (a Slurmctld-side job submit plugin)
- **How to verify (effective config)**:

```bash
scontrol show config | egrep -i 'JobSubmitPlugins|PluginDir'
```

Example expected output includes:
- `JobSubmitPlugins = list_accounts,lua`
- `PluginDir = /usr/lib64/slurm`

## Summary table (high-signal)

| Scope | What it checks | What happens | Notes / user-visible behavior |
|---|---|---|---|
| **Della (global)** | Missing `--time` | **Reject** | Returns `ESLURM_INVALID_TIME_LIMIT` |
| **Della (global)** | `--time` too large | **Reject** | Hard cap via `VLONG_MINS` in Lua |
| **Della (global)** | Submitting from personal `/scratch/gpfs/<netid>` | **Reject** | Must use faculty-owned fileset |
| **Della (global)** | Partition set to `gputest` or `gpu` directly | **Reject** | Forces users into normal GPU flow rather than raw partition names |
| **Della (global)** | `--mem=0` or `--mem-per-cpu=0` | **Reject** | Invalid “unlimited memory” request |
| **PLI** | Partition includes both `pli*` and `ailab*` | **Reject** | Must choose one partition family |
| **PLI** | No GPUs requested (any GPU-less job on PLI) | **Reject** | “requesting PLI nodes but not allocating GPUs” |
| **PLI** | `--time` > 3 days for PLI partitions | **Reject** | PLI time policy is stricter than default |
| **PLI** | Multinode GPU job not “dense enough” | **Reject** | Anti-fragmentation: forbids e.g. 2 nodes × 4 GPUs |
| **PLI** | `pli-cp` QoS time > 8 hours | **Reject** | Special “core priority” debugging QoS |
| **PLI** | Single-GPU jobs | **Mutate** | Adds an exclude list to reserve some nodes for multi-GPU jobs |
| **ailab** | Not requesting GPUs | **Reject** | ailab nodes require GPU allocation |
| **ailab** | CPU over-allocation per GPU | **Reject** | Enforces ≤ 8 CPU cores per GPU (per node) |
| **ailab** | Using account `pli` | **Reject** | Must use PI account, not PLI |

---

## Della restrictions (global / general)

These apply regardless of partition (unless a special-partition early-return overrides the later logic).

### Time limits

- **`--time` is required**: if `job_desc.time_limit` is missing, the submit plugin rejects the job with `ESLURM_INVALID_TIME_LIMIT`.
- **Cluster-wide max `--time`**: the plugin rejects if `job_desc.time_limit > VLONG_MINS`, where `VLONG_MINS = 8641` minutes (≈ 6 days).

### Workdir restriction: personal `/scratch/gpfs/<netid>`

If submission happens from a personally-owned scratch area (pattern check on `job_desc.work_dir`), the job is rejected with:
- “You cannot submit jobs from personally owned /scratch/gpfs directory … please start using a faculty owned fileset.”

### Disallowed partitions at submit time

The plugin rejects submissions that directly specify:
- `-p gputest`
- `-p gpu`

This is separate from the later logic that routes GPU jobs into an appropriate GPU QoS/partition flow.

### Memory request sanity checks

Jobs requesting “unlimited” memory are rejected:
- `--mem=0` (mapped to `pn_min_memory == 0`)
- `--mem-per-cpu=0` (`min_mem_per_cpu == 0`)

### Walltime bins (general GPU flow)

**This table is the single source of truth for the bins.** [priority.md](priority.md) and
[values.md](values.md) link here rather than repeating it — update this section only.

For jobs detected as GPU jobs (`is_GPU_job()`), the plugin builds the QoS name in three steps:

1. `getQOS(job_desc.time_limit)` returns a base tier from walltime.
2. `vlong` is remapped to `long` — GPU jobs only, so there is no `gpu-vlong`.
3. `job_desc.qos = "gpu-" .. <base>`.

Time bins, from the constants at the top of the Lua file:

| Constant | Value (min) | `--time` | Base tier | GPU QoS |
|----------|-------------|----------|-----------|---------|
| `TEST_MINS` | 60 | ≤ 1 h | `test` | `gpu-test` |
| `SHORT_MINS` | 1441 | ≤ 24 h | `short` | `gpu-short` |
| `MEDIUM_MINS` | 4321 | ≤ 72 h | `medium` | `gpu-medium` |
| `VLONG_MINS` | 8641 | ≤ 6 days | `vlong` → `long` | `gpu-long` |

Anything above `VLONG_MINS` is rejected earlier with `ESLURM_INVALID_TIME_LIMIT`. `LONG_MINS`
(2880) is commented out in the source, so no job is assigned a bare `long` tier by walltime.

The bins are one minute past the round hour figure (1441, 4321) rather than exactly on it, so a
24-hour request stays in `short` with a minute of slack.

There is **no `gpu-vlong`**: the `vlong → long` remap applies to GPU jobs only, so a CPU job keeps
the bare tier name (`test`, `short`, `medium`, `vlong`) while a GPU job of the same walltime becomes
`gpu-long`. Because the tier decides both the QoS priority and the per-user GPU ceiling, `--time` is
the main knob a user has over both — see [priority.md](priority.md) and [values.md](values.md).

The `VLONG_MINS` cap is enforced by the plugin and is **stricter than some partition `MaxTime`
values** (`ailab` advertises 15 days). The submit filter wins: the job is rejected before it is ever
queued.

### `gpu-test` partition redirect

A job that lands in `gpu-test` also has its **partition rewritten to `gputest`** — unless it is on
`mig`, `grace`, `rtx6000`, `ailab`, `ailab-p`, or `hackathon`, which keep the partition they asked
for. This is how short GPU test jobs reach the test hardware without the user naming it, and it is
why users must **not** pass `-p gputest` themselves (that is rejected outright — see *Disallowed
partitions at submit time* above). The correct user-facing instruction is "request ≤ 1 hour", not
"submit to gputest".

---

## PLI restrictions (pli*, incl. pli, pli-c, pli-p, pli-lc)

PLI logic is handled in the `if string.match(job_desc.partition, "pli") then ... end` block.

### Must not combine `pli*` and `ailab*` partitions

If the partition string matches both `pli` and `ailab`, the job is rejected:
- “You cannot specify both pli and ailab partitions!”

### PLI time limits + QoS selection

- **PLI partitions require `--time` ≤ 3 days** (else rejected).
- The plugin sets PLI QoS depending on the specific partition:
  - **`pli-c`**: derives `pli-short|pli-medium` etc via `set_plic_qos()`, and also adjusts account to `pli` if not `cses`
  - **`pli-p`**: forces `job_desc.qos = "pli-high"`
  - **`pli`**: sets `job_desc.qos = "pli-low"` (subject to time policy)
  - **`pli-lc`**: sets `job_desc.qos = "pli-lc"`

Special case:
- **`pli-cp` QoS**: enforces `--time` ≤ 8 hours (480 minutes), else rejected.

### PLI requires GPUs

Any PLI job that does not request GPUs is rejected:
- “requesting the PLI nodes but not allocating GPUs”

### PLI anti-fragmentation policy (the “prefer 1×8 over 2×4” rule)

For multinode PLI GPU jobs (`get_total_NODES(job_desc) > 1`), the plugin rejects if the total GPU request is too small for the node count.

The code comment explains the threshold:
- “minimum GPUs for NODES is (N-1)*8 +1 == N*8 - 7”

In effect, with **8 GPUs per node**, the minimum total GPUs required for `N` nodes is:
\[
\text{total GPUs} \ge 8\cdot(N-1) + 1
\]

Concrete examples:
- **2 nodes**: must request **≥ 9 GPUs** total (so **2×4=8 is rejected**, pushing users to **1×8** instead)
- **3 nodes**: must request **≥ 17 GPUs** total

This is a **submission-time rejection** (not a scheduler preference).

### PLI single-GPU placement (node exclusions)

If a PLI job requests exactly **1 GPU**, the plugin mutates the request by adding an **exclude list** (via `PLI_exclude(job_desc)`) to keep certain nodes reserved for multi-GPU jobs. Currently, 40 percent of the nodes are reserved for multi-GPU jobs.

---

## ailab restrictions (`ailab`)

ailab logic is enforced in `ailab_check(job_desc)`, invoked when `job_desc.partition` matches `ailab`.

### ailab requires GPUs

If the job targets ailab nodes but does not request GPUs, it is rejected:
- “requesting the AI Lab nodes but not allocating GPUs”

### CPU per GPU cap (per node)

ailab enforces **≤ 8 CPU cores per GPU per node**.

Implementation details:
- `CORES = get_node_CPUs(job_desc)` (derived from tasks × cpus-per-task; intended to be per-node)
- `GPUS = get_node_GPUS(job_desc)` (derived primarily from `tres_per_node` / `tres_per_job`)
- Reject if `CORES > GPUS * 8`

### Account restriction

ailab rejects jobs using the `pli` account:
- “requesting the AI Lab nodes but using the PLI account … use your PI account”

# Additional Information

## How the pieces connect (RPC → plugins → files)

1. **`sbatch`** sends a **submit RPC** to **`slurmctld`**. Submit-time policy runs **on the controller**, not inside the `sbatch` binary.

2. **`slurmctld`** walks **`JobSubmitPlugins`** **left to right** (order in `slurm.conf`). Each name `N` loads **`$PluginDir/job_submit_N.so`** and runs its job-submit hook. Example: `list_accounts` → **`/usr/lib64/slurm/job_submit_list_accounts.so`**.

3. **Lua** is another entry in the same list: if `lua` is present, **`/etc/slurm/job_submit.lua`** is driven by **`job_submit_lua.so`**, and Slurm calls **`slurm_job_submit(job_desc, part_list, submit_uid)`** in that file. That runs **in plugin order relative to `list_accounts`** (e.g. with `list_accounts,lua`, list-accounts runs first).

4. **Why `strings(1)` on a `.so` matches user errors:** C plugins embed message templates in the shared object. The plugin prints them when it rejects or warns; the text is not stored in a separate message catalog at runtime.

5. **Stock errors** (e.g. “Invalid account or account/partition combination specified”) usually come from **Slurm’s association / accounting checks**, not from `job_submit.lua`. A single failed submit can return **both** a **plugin-specific** line and a **core Slurm** line.

All of this happens **before** the job is scheduled or allocated; failures return over the same RPC `sbatch` already opened.

## Inspecting compiled plugins (`.so`)

```bash
strings /usr/lib64/slurm/job_submit_list_accounts.so | rg -i 'account|ERROR'
rg -a 'You have to specify an account' /usr/lib64/slurm/*.so
```

`rg` may report `binary file matches` without printing lines; use `strings` piped to `rg`, or narrow to one `.so`.