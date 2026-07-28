---
name: cluster
description: >-
  Shared-HPC (Slurm) conventions — never compute on a login node, size requests
  to measurement, submit instead of backgrounding, read the real allocation not
  the node, benchmark hardware rather than assuming bigger is faster, and record
  machine-specific facts as a dated snapshot. Use before running anything heavier
  than a shell query on a cluster, or when writing/sizing/submitting batch jobs.
---

# Cluster usage (shared HPC, Slurm)

A cluster is a **shared** resource: every over-padded request, every process on a
login node, and every idle-but-held GPU is taken from someone else. These
conventions are portable across machines. The *facts* about any particular
cluster (partition names, GPU types, filesystems, hostnames) are not portable —
see [Recording machine-specific facts](#recording-machine-specific-facts).

## The one hard rule: never run compute on a login node

Login nodes are shared, tiny (often 1–2 usable cores), and already loaded by
everyone else's shell work. Running an analysis, plot, or model there is both
slow *and* degrades a resource everyone depends on.

- ✅ Login node is fine for **cheap shell queries**: `git`, `ls`, `grep`, reading
  logs, editing files, `squeue` / `sinfo` / `sacct`, syntax checks
  (`python -m py_compile`).
- ❌ Never run compute there — **including backgrounded**. `nohup python … &`,
  `screen`, and `tmux` do not make it acceptable; the constraint is the node, not
  the foreground/background distinction.
- ✅ Submit instead. `sbatch` for anything real; `srun`/`salloc` for a quick
  interactive test. Never a bare `python` on a heavy script.

Check where you are before running anything: `hostname`, `nproc`, `uptime`
(load average). A load average far above the core count means the node is
already saturated.

## Sizing requests

- **Size to measurement, not habit.** Derive every resource number from a
  profile of a real run: peak RSS for `--mem`, measured wall time (plus modest
  headroom) for `--time`, the parallel width the code actually uses for
  `--cpus-per-task`. Read them back after the fact:

  ```bash
  sacct -j <jobid> --format=JobID,JobName,Elapsed,MaxRSS,ReqMem,AllocCPUS,State
  ```

- **Don't pad "to be safe."** Oversized requests sit longer in the queue *and*
  block other users; a job asking for 10× the memory it uses is fragmenting the
  cluster for no benefit. Right-size, then add headroom only where a failure is
  expensive (long jobs without checkpoints).
- **Prefer many small jobs (or an array) over one giant one.** Small jobs
  backfill into gaps and start sooner. Use `sbatch --array` rather than a
  hand-rolled submission loop.
- **Only request a GPU if the code uses one.** Analysis, plotting, and data
  wrangling belong on a CPU partition.

## Read the real allocation, not the node

Slurm gives you a *subset* of a node. Code that inspects the machine will
oversubscribe and thrash — hurting your job and every co-tenant on that node.

```python
import os

# ✅ what Slurm actually gave this process
n_cpus = len(os.sched_getaffinity(0))

# ❌ reports every core on the node regardless of the allocation
n_cpus = os.cpu_count()
```

The same applies to derived defaults: thread-pool sizes, `n_jobs`, BLAS/OpenMP
thread counts (`OMP_NUM_THREADS`), and any "use all cores" library setting
should follow the allocation, not the hardware. Prefer `$SLURM_CPUS_PER_TASK` /
`$SLURM_MEM_PER_NODE` in shell scripts, with a sane fallback for local runs.

## Choose hardware by benchmark, not by prestige

**Do not assume the biggest accelerator is the best one for the job.** Top-end
cards win on problems large enough to saturate them; below that threshold a
mid-tier card can match or beat them, while being far less contended and much
faster to schedule. The same holds for CPU: more cores help only up to the point
where the workload actually scales.

- Benchmark the *actual* workload on each candidate resource before committing
  to it — a short pilot job is cheap relative to weeks of production.
- Record the measurement (throughput per device, scaling curve) next to the
  configuration it justifies, so the choice is auditable later.
- Queue time counts. A resource that is 20% faster but takes days to schedule is
  slower end-to-end.

## Don't hoard scarce resources

- **Match concurrency to the deadline, not to what is momentarily idle.** If six
  jobs finish comfortably in time, do not run twenty because the queue looks
  empty right now.
- **Never take a large fraction of a scarce shared pool.** A handful of the
  cluster's rarest accelerators is not yours to occupy for a week; prefer the
  plentiful resource sized to the deadline.
- **Release what you're not using.** Cancel dead jobs (`scancel`), don't hold
  interactive allocations open while you think, and don't keep an
  `salloc`-backed shell alive overnight.
- **Preemptable partitions are for requeue-safe work only.** They are the polite
  way to use idle capacity — but only submit there once checkpoint/restart works
  and the job is submitted with `--requeue`. Untested restart plus preemption
  means silently losing days.

## Submitting

- **Keep submission in a script, not in shell history.** A `bin/submit_*.sh`
  wrapper that assembles the `sbatch` command makes the resource choices
  reviewable, diffable, and reproducible.
- **Dry-run by default.** Have wrappers print the command they would run and
  require an explicit `--submit` to queue it. Cheap insurance against a typo
  that queues 500 jobs.
- **Validate parameters before submission.** Check partition names, GRES
  strings, and account/QOS at launch time so a mistake fails immediately rather
  than after a job sits pending for hours (or dies at minute one of a 7-day
  allocation).
- **Chain long work into segments.** Where a job exceeds the partition's wall
  limit, submit a dependency chain (`sbatch --dependency=afterany:<jobid>`) of
  checkpoint-resuming segments rather than asking for an exception.
- **Always write checkpoints** for anything longer than an hour, and make resume
  the default path — nodes fail, jobs get preempted, and wall limits arrive.
- **Redirect output somewhere durable and named** (`--output=logs/%x-%j.out`) so
  a failure is diagnosable without re-running.

## Recurring jobs

If the cluster enables `scrontab`, use it. If it does not, use a
**self-resubmitting chain**: each job queues its successor with
`sbatch --begin=now+<interval>` **before** doing its work — so a failing run
cannot break the chain — and self-terminates on an explicit end-date guard.
Verify first that compute nodes are permitted to submit jobs; some sites forbid
it. Always include a stop condition; an unbounded chain is a resource leak that
outlives your interest in it.

## Viewing a service (dashboard, notebook) remotely

Compute nodes generally have no public web path. Port-forward through the login
node instead of exposing anything:

```bash
ssh -fN -L <local-port>:<compute-node>:<remote-port> <user>@<login-host>
# then browse http://localhost:<local-port>/
```

- Use the **externally routable** login hostname. `hostname -f` on a login node
  often returns an internal-only name that will not resolve from your laptop.
- Bind the service to the compute node's interface (or `0.0.0.0`) and pick a
  high, non-privileged port; check the login node can reach it (`curl -I`)
  before debugging the tunnel.
- Run the server as a **job**, not on the login node — the rule above still
  applies to long-lived servers.

## Storage and environment gotchas

- **Know which filesystems are writable, which are purged, and which are
  quota-tight** before writing anything large. Scratch is often fast but
  auto-purged; `/home` is usually small; project space is usually the right
  target. Don't trust `df` blindly on network filesystems — it can misreport.
- **Compute nodes may not share your local or session-temporary directories.**
  Stage helper scripts and inputs onto shared project storage before submitting;
  anything under a session-local `/tmp` will not exist on the node.
- **`conda`/`mamba` are shell functions interactively.** Batch scripts must
  resolve the real binary or source the profile script explicitly, or activation
  silently fails under `#!/bin/bash` with no login shell.
- **Build GPU environments where a GPU is visible.** Dependency resolution can
  pick CPU-only builds when no device is present; create/verify the environment
  in a GPU job.
- **Avoid heredoc-into-`conda run`-style invocations** — stdin is not always
  forwarded and the command can silently no-op. Write the script to a file and
  run the file.

## Recording machine-specific facts

Portable norms live here; the specifics of a given cluster belong in a **project
or machine-local skill** (e.g. `.claude/skills/cluster-<machine>/SKILL.md`) that
extends this one. Keep that file:

- **Dated.** Open with a snapshot line — "Verified YYYY-MM-DD" — because
  partitions get added, nodes drain, and hostnames change. Update the date
  whenever you re-verify.
- **Self-verifying.** End with the exact login-node-safe queries that regenerate
  every fact in it, so it can be refreshed in one pass rather than trusted
  blindly.
- **Scoped to facts and local policy**: partitions and wall limits, account/QOS,
  hardware inventory, filesystem layout and quotas, login/bastion hostnames,
  scheduler quirks, and any measured benchmarks that justify the current
  configuration.

### Discovery queries (login-node-safe)

```bash
sinfo -o "%.20P %.6c %.12l %.28G %.6D %.12T"        # partitions, cores, wall limit, GRES, state
sinfo -o "%.28G" -h | grep -i gpu | sort -u          # GPU types present
sacctmgr -n show assoc user=$USER format=Account,Partition,QOS%20
squeue -u $USER -o "%.10i %.18P %.10j %.8T %.10M %R"  # my jobs + why pending
sacct -u $USER -S <YYYY-MM-DD> --format=JobID,JobName,Partition,Elapsed,MaxRSS,AllocCPUS,State
scontrol show partition <name>                       # limits, allowed accounts, defaults
scontrol show node <name>                            # cores, memory, GRES, current load
```

Note that `PrivateData` may be enabled, in which case `squeue` shows only *your*
jobs — partition load must then be inferred from `sinfo` node states rather than
from other users' queued work.
