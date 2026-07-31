# Troubleshooting

## "You have to specify an account" / "Invalid account or account/partition combination"

The allowed-account message comes from **`job_submit_list_accounts.so`** (`list_accounts` in
`JobSubmitPlugins`), not from `job_submit.lua`. A second line is usually Slurm's stock association
error. Confirm associations, then see [submit-restrictions.md](submit-restrictions.md).

```bash
sacctmgr show assoc where User=<user> format=Cluster,Account,User,Partition,QOS%200
```

## "QOSMaxGRESPerUser"

The user hit the GPU ceiling for their QOS. Compare `squeue -u <user>` against live QOS limits
(`sacctmgr show qos`, or [values.md](values.md) for the recorded snapshot).

## "Priority"

Queued behind higher-priority work. Break the score down with `sprio -j <jobid>` and
`sshare -u <user>` — see [priority.md](priority.md). Fairshare often matters more than QOS, and
priority does not fix node fragmentation, so also check `shownodes` and free GPUs per node.

## "Resources"

Not enough free resources right now: `gfree` and `shownodes -p <partition>`.

## Node in "drain" or "down" state

```bash
scontrol show node <node> | grep -i reason
```

## Who is using resources

```bash
squeue -p <partition> -t R -o "%.10i %.9P %.20j %.8u %.2t %.10M %.6D %.4C %.8b %R"
```
