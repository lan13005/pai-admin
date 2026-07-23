# PLI partition reference (user-facing model)

Condensed from the PLI Della guide ([Notion](https://zinc-scale-b3f.notion.site/The-Della-cluster-and-the-PLI-partition-a3526cf557334124903964a3fa529f68)) by Yihe Dong. Use this when answering **who uses which PLI partition**, **shared-GPU misconceptions**, **storage**, or **user best practices**.

**Do not treat QOS numbers here as live truth.** Re-verify with `sacctmgr show qos`, `scontrol show partition`, and `job_submit.lua`. Hard submit rules: **`01-job-submit-restrictions.md`**. Ops tables: **`SKILL.md` → Reference values**.

---

## When to consult this file

| Question type | Look here |
|---------------|-----------|
| Which PLI partition for proposal vs core vs priority users? | Partition family |
| Why is my PLI queue empty but jobs still wait? | Partitions ≠ fixed GPU pools |
| CPU/mem per GPU, interactive limits, `-N 1` | Best practices |
| Single-GPU / array jobs on PLI | Single-GPU jobs |
| `/scratch` vs TigerData | Storage |
| Who to email for unresolved PLI compute asks | Support |

---

## Partition family

In most cases users submit to **`pli`**, **`pli-lc`**, or **`pli-c`** (partition names, not Linux groups).

| Situation | Typical partition |
|-----------|-------------------|
| Access via PLI compute proposal (named account) | `pli` (default) or `pli-lc` |
| PLI members focused on large language models | `pli-c` |
| Rare, selected high-priority projects | `pli-p` |

Users may still use non-PLI Della GPUs. For quick tests, prefer Della **`gputest`** / short general-GPU jobs (&lt; 1 h). PLI has **no test partition**; wait times are expected. PLI is primarily for `sbatch`, not interactive testing.

**Age priority:** only the **10 oldest actively waiting jobs per user** accrue age-based priority.

Submit with partition (`-p` / `-P`) and account (`-A`). QOS is usually chosen automatically from affiliation and walltime (see Lua / cluster setup below).

---

## Cluster setup (intent + QOS mapping)

Snapshot of the user guide’s intent table. **Priorities and Max\*GPU limits drift — confirm live.**

| Partition | Who | QOS | Time (guide) | Usage intent |
|-----------|-----|-----|--------------|--------------|
| `pli-p` (“priority”) | Rare selected members/projects | `pli-high` | — | Very large / high-priority jobs (e.g. big multi-node training). Rare; short allocation windows when approved. |
| `pli-c` (“core”) | PLI members on large LLMs | `pli-short` | 24 h | Pre-train / fine-tune LLMs |
| | | `pli-medium` | 72 h | Longer core jobs |
| | | `pli-cp` | 8 h | Short debugging; guide mentions a rolling GPU-hour budget per user |
| `pli` (“campus”) | Proposal-granted access | `pli-low` | 72 h | Work that doesn’t fit regular Della GPU / other clusters; large AI models |
| `pli-lc` (“large campus”) | Larger campus allocation | `pli-lc` | 72 h | Same intent as campus, larger-campus tier |

### Column meanings (user-guide framing)

- **QOS:** Quality of service; usually set from affiliation + job duration (submit plugin).
- **Prio:** Higher base QOS priority → earlier in multifactor QOS term (fairshare/age still matter).
- **Max Grp GPU:** Cap on GPUs in use under that QOS **cluster-wide**, not “GPUs owned by the partition.”
- **`pli-p`:** Reserved for outstanding, proven recipes; short windows; not for deadline crunching by default.

### Partitions ≠ fixed GPU pools

A partition is better thought of as a **queue with rules**, not a dedicated set of GPUs. Physical PLI GPUs are shared across PLI partitions. An empty `pli` queue can still wait if other PLI partitions hold the nodes. Limits (e.g. Max Grp GPU) constrain how much of the shared pool a QOS/partition family can occupy.

---

## Best practices (user guidance)

- **CPU / memory per GPU:** Guide assumes ~8 GPUs/node, ~1024 GB mem, ~96 CPUs → request **≤ ~128 GB mem and ≤ ~12 CPUs per GPU** (confirm node shape with `shownodes` / `scontrol show node`).
- **Interactive jobs:** Prefer ≤ **2** concurrent interactive jobs, ≤ **2 hours** each.
- **Same-node GPUs:** Use `-N 1` when you need GPUs on one node (fast intra-node links) instead of scattering across nodes.
- **Efficiency:** Users can check GPU efficiency with `reportseff` (or mail-type end summaries). Inefficient GPU jobs may be killed.
- **Do not submit on others’ behalf** outside research collaborations.

---

## Single-GPU jobs

`pli` / `pli-c` / `pli-lc` are aimed at multi-GPU training and have high per-user GPU ceilings. Guidance:

1. Prefer **regular Della GPU** partitions for single-GPU work when possible.
2. Large batches of single-GPU work on PLI → use **array jobs**.
3. Cap concurrent array tasks (guide example: **32**) and keep **`della-j*`** free for multi-GPU (Lua also applies single-GPU excludes — see **`01-job-submit-restrictions.md`**).

---

## Storage

| Location | Role |
|----------|------|
| `/scratch` | High-performance, low latency; active training / development. **Limited.** |
| TigerData `/tigerdata/pli/data` | Longer-term checkpoints and artifacts not in active use |

TigerData is reachable from head nodes such as **`della9`**, not from **`della-pli`**.

---

## Support

For compute requests/questions that cannot be resolved otherwise: **`pli-rse@princeton.edu`** (prefer this over messaging individuals).

Upstream user docs often also point at the Della tutorial and Princeton RC SLURM knowledge base.
