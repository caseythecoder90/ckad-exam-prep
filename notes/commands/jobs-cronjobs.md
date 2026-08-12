# Jobs & CronJobs

## Job (runs once to completion)

```bash
k create job pi --image=perl $do -- perl -Mbignum=bpi -wle "print bpi(100)" > job.yaml
k create job pi --image=perl -- perl -Mbignum=bpi -wle "print bpi(100)"     # imperative
```

## CronJob (recurring on a cron schedule)

```bash
k create cronjob backup --image=busybox --schedule="0 */6 * * *" $do -- echo backup > cron.yaml
k create cronjob backup --image=busybox --schedule="*/5 * * * *" -- /backup.sh
```

## The `--` separator (two gotchas that produce wrong YAML)

**Multi-command in `--` → always `sh -c "cmd1 && cmd2"`, never bare `&&`.** The container command is an argv list with no shell to interpret `&&`, so your local shell splits the line and runs the second half on your own machine.

```bash
# WRONG — creates a job running only `cmd1`, then runs `cmd2` locally
k create job x -n batch --image=busybox -- cmd1 && cmd2

# RIGHT
k create job x -n batch --image=busybox -- sh -c "cmd1 && cmd2"
```

**Every kubectl flag goes before `--`.** Flag parsing stops at the separator, so `$do` after it is swallowed into the container command — the Job is created for real with junk args instead of printing YAML.

```bash
k create job x -n batch --image=busybox $do -- sh -c "cmd1 && cmd2" > job.yaml
```

Shape: `k create job <name> <all flags> -- sh -c "…"`. Full detail in [`imperative.md`](imperative.md).

## Trigger a CronJob manually

Creates a one-off Job from the CronJob's pod template — useful for testing without waiting for the next schedule tick.

```bash
k create job --from=cronjob/backup manual-backup
```

## Inspect

```bash
k get jobs
k get cronjobs                       # short: cj
k describe job <name>
k describe cronjob <name>
k logs job/<name>                    # logs from the pod the job ran
```

## Delete

```bash
k delete job <name>
k delete cronjob <name>              # stops future scheduled runs
```

## Key spec fields

- `spec.completions` — total successful pod completions required.
- `spec.parallelism` — how many pods run in parallel.
- `spec.backoffLimit` — retries before the Job is marked Failed (default 6).
- `spec.activeDeadlineSeconds` — hard timeout for the whole Job.
- `spec.ttlSecondsAfterFinished` — auto-delete completed/failed Jobs after N seconds.

CronJob adds:

- `spec.schedule` — cron expression.
- `spec.concurrencyPolicy` — `Allow` (default), `Forbid`, `Replace`.
- `spec.successfulJobsHistoryLimit` / `failedJobsHistoryLimit` — how many old Jobs to retain.

## See also

- `imperative.md` — full `k create job` / `k create cronjob` syntax
- `debugging.md` — `k logs job/<name>` and reading describe output
