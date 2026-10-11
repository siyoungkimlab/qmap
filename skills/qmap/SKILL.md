---
name: qmap
description: Submit, monitor, speed up and repair batch jobs on MSKCC Lilac (LSF) with the user's qmap tool (~/qmap on lilac). Use when writing or editing a qmap job file (#!/usr/bin/env qmap, `!` lines, #Lilac_* templates), submitting array jobs, checking job progress or ETA, figuring out why jobs are pending or slow, getting more jobs to run at once, or rerunning failed or killed tasks on lilac.
---

# qmap on Lilac

qmap lives on lilac at `~/qmap` (`bin/qmap`, `templates/*.args`, `README.md`,
and this skill in `skills/qmap/`).

- On lilac itself (hostname `lilac-*`), run the commands directly. If `bjobs` is
  not found, run `source /etc/profile` first.
- From another machine (e.g. the user's laptop), wrap them:
  `ssh -o BatchMode=yes lilac 'source /etc/profile >/dev/null 2>&1; ...'`.
- Don't edit the boonza source (`~/boonza`); that is done in another session.

## Job files

```python
#!/usr/bin/env qmap
#Lilac_CPU_6h            # a bare #Name is a template; key=value lines come after it
#mem=8                   # GB per slot (LSF rusage[mem]), so 8 cores x 8 = 64 GB
#name=myjob
#logdir=logs
#conda=boonza
#concurrency=200         # the array's %N: how many tasks may run at once

for x in items:
    ! cmd1 ${x}          # every `!` in one loop iteration becomes ONE task, joined with &&
    ! cmd2 ${x}          # so cmd2 is skipped if cmd1 fails
```

- The Python runs on the login node to build the task list; `${var}` is filled in from Python and shell-quoted.
- `./job.py --dry-run` prints the `#BSUB` header and the tasks without submitting anything. Run it before every submission.
- `./job.py` submits. The task list is frozen in `.qmap/<name>-<timestamp>/inputs.txt`, where line i is task i, plus `job.sh` and `jobid`. Editing the .py afterwards does not change a submitted job.
- Header keys: scheduler, queue, time, cores, gpu, mem, constraint, concurrency, dependency, bundle, repeat, conda, module, name, logdir, export, setup, directive. An unknown key stops the run.
- `--bundle N` packs N inputs into each task, for many short inputs. `--dependency ok:ID|any:ID` chains jobs.
- Check before resubmitting: qmap does not prevent duplicates. Two submissions of the same file mean every system runs twice, writing into the same folders.

## Templates (`qmap templates`, `qmap show NAME`)

| Template | Queue | What it's for |
|---|---|---|
| `Lilac_1GPU_4h` | gpuqueue, 8 cores, 1 GPU, 4 h | MD on any free card except A100/V100 |
| `Lilac_A100_1GPU_4h` | gpuqueue, 12 cores, A100, 4 h | MD on A100 hosts |
| `Lilac_CPU_6h` | cpuqueue, 8 cores, 6 h | analysis without a GPU, on regular hosts |
| `Lilac_A100_CPU_6h` | cpuqueue, 8 cores, A100 hosts, 6 h | analysis without a GPU, on the often-idle A100 hosts |

## How Lilac schedules (why jobs pend while GPUs sit idle)

1. **Keep walltime at 6 h or less (`#time=6h`).** Lab-owned host pools (Pavletich lj/ll04/ln07-10, Pe'er ll01-03, Quaid ly, ...) lend their idle hosts only to jobs with a run limit of 360 min or less (`bresources -g -l` shows `LOAN_POLICIES ... DURATION[360]`). `#time=48h` locks you out of them.
2. **A100 and V100 hosts are exclusive.** `lshosts -l lj01` shows `RESOURCES: (gpu !A100)`, so LSF sends a job there only if it explicitly asks for `A100`, and that includes CPU-only jobs. Plain templates never land on lj/ll/lc/lz/lk/ld/lv. Use the A100 templates or `--constraint A100`.
3. **There are no CPU-only nodes.** `cpuqueue` uses the free cores of the GPU hosts. Analysis that needs no GPU (boonza sites/sitemap/solute) should go to cpuqueue so it doesn't hold GPUs.
4. **Per-host-group slot limits** (`limit1107` = 56, `limit1103` = 112, ...) and the array's own `%N` concurrency also cap how many tasks run at once.
5. **`boonza md` initialises CUDA even when its run is already finished.** A rerun of a whole pipeline (swim → md → ...) therefore needs a GPU. For analysis-only reruns, write a job file without swim/md.

To get many tasks running, split the work: part on regular hosts (`Lilac_CPU_6h`) and part on A100 hosts (`Lilac_A100_CPU_6h`). Or move pending tasks over with `bmod` (below). On 2026-10-11 this took fpocket200 from 48 tasks running at once to about 300.

## Monitoring

```bash
bjobs -A JOBID                     # one-line array summary: PEND DONE RUN EXIT
bjobs -noheader -o "jobindex stat run_time exec_host" JOBID
bjobs -p3 "JOBID[i]"               # pending reason broken down by host: the key diagnostic
bjobs -l -p3 "JOBID[i]"            # same, with host names
bjobs -l "JOBID[i]" | grep -E "MAX MEM|TERM_|Exited"   # memory use and why a task died
busers -w $USER; bqueues -w; bhosts -gpu -w             # load overall
```

- **ETA:** measure the time per task from finished tasks (`run_time` of DONE ones, or output file mtimes per step), then remaining ≈ pending × mean ÷ running at once, plus the slowest task.
- `TERM_MEMLIMIT` means the task ran out of memory. The usual cause in boonza is `sitemap traj` without `--jobs N`: it defaults to `os.cpu_count() - 2`, which counts the whole node (80-128 cores), not your slots. Always pass `--jobs` equal to the core count.
- LSF writes the `.out` log only when a task ends. For tasks still running, look at the output files themselves.

## Changing jobs in place (no resubmission)

```bash
bmod -W 6:00 JOBID                                   # walltime of the PENDING tasks (unlocks the loaner pools)
bmod -R "select[A100] span[hosts=1] rusage[mem=8]" "JOBID[a-b]"   # move pending tasks to A100 hosts
                                                     # (-R replaces the whole string: keep span and rusage)
bmod -J "%400" JOBID                                 # raise the array's concurrency limit
```

`bmod` only affects tasks that haven't started; running tasks keep their old settings.

## Killing (this LSF has no `bkill -stat`)

```bash
bkill JOBID                                          # whole array
bkill "JOBID[5]"                                     # one task
bkill $(bjobs -noheader -o "jobid jobindex stat" JOBID | awk '$3=="PEND"{printf "\"%s[%s]\" ", $1, $2}' | xargs)   # pending tasks only
```

`bkill` may be blocked for Claude by the permission check. If so, hand the exact command to the user.

## Rerunning a subset safely

1. Save the state first: `bjobs -a -noheader -o "jobindex stat run_time" JOBID > .qmap/state.txt`.
2. Kill the tasks you are replacing.
3. Rebuild the list from what was actually killed (EXIT with run_time 0), not from an earlier pending list, because tasks start in the meantime. Map index → workdir through line i of `.qmap/<run>/inputs.txt`.
4. Check that none of those workdirs overlaps a task that is still running.
5. Have the new job file read that list, dry-run it, then submit once.

## Caches and restarts (boonza)

- `sitemap traj` writes `OUT/sites.pkl` only after all frames are done, atomically. A crash mid-frames leaves no cache and a rerun recomputes; a crash later reuses the cache.
- Use `--recompute` only when boonza's site-finding code or the trajectory has changed.
- MD resumes from `md/checkpoint.chk`. `md/status.json` `"outcome": "target_reached"` means the MD is finished.
