# Slurm configuration on Della

Where operator-facing config lives, and how to read the **effective** values rather than the files.

## Files

Under **`/etc/slurm/`**:

| File | Contents |
|------|----------|
| `slurm.conf` | Partitions, nodes, `AccountingStorage*`, `JobSubmitPlugins`, priority weights |
| `slurmdbd.conf` | Accounting DB bridge (usually on the DB host, not the login nodes) |
| `gres.conf` | GPU definitions |
| `cgroup.conf` | Per-job resource containment |
| `topology.conf` | Network topology used for placement |
| `plugstack.conf` | SPANK plugin stack |
| `job_submit.lua` | Lua submit filter — rules in [submit-restrictions.md](submit-restrictions.md) |

## Read the effective config

The controller's running config can differ from the files on any given host, so prefer `scontrol`:

```bash
scontrol show config | head -30
scontrol show config | rg -i 'PriorityWeight|PriorityType|PriorityFlags|PriorityMax'
scontrol show config | egrep -i 'JobSubmitPlugins|PluginDir'
scontrol show config | rg -i 'AccountingStorageEnforce|PriorityUsageResetPeriod'
scontrol show partition <partition>
```

Compiled submit plugins live in `PluginDir` (typically `/usr/lib64/slurm`) as
`job_submit_<name>.so`. Inspecting them: [submit-restrictions.md](submit-restrictions.md).
