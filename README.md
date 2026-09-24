# qmap

Map one command over a list of inputs, as an LSF or Slurm job array.

One command, two ways to use it: **`qmap job.py`** runs a Python job file,
which is the better fit when working out the inputs takes real code, and
**`qmap [flags] -- cmd`** is the one-liner form. A job file is recognised by
being a `.py` file among the arguments; the flag form always has a `--`
payload, so the two never collide.

Every array job has the same three parts: a list of inputs, a command line to
run on each one, and a block of scheduler directives. Only the directives
differ between clusters. `qmap` writes those, templates your command line with
Python's `str.format` syntax, and submits. It knows nothing about any
particular program.

`--scheduler local` drops the directives and runs the tasks on the machine
you are sitting at, so the same command line works on a laptop and on a
cluster.

**Nothing is set for you.** An account, queue, walltime or core count appears
in the generated script only because you asked for it — on the command line,
or through a template you registered. There is no implicit cluster default.

## Install

```bash
git clone https://github.com/siyoungkimlab/qmap.git ~/qmap
export PATH="$HOME/qmap/bin:$PATH"
```

No dependencies: bash 3.2 and any Python 3, so the same clone runs on a Mac
and on a login node. Job files need Python; the flag form does not.

## Job files: `qmap job.py`

Ordinary Python, plus a header of `#key=value` lines and `!` lines that run a
shell command:

```python
#workload_manager=lsf
#time=24:00
#core=4
#gpu=1

import glob
ifiles = glob.glob('*/*/*.mae')

! mkdir -p step3

for ifile in ifiles:
    workdir = ifile.split('/')[1]
    ! boonza md ${ifile} --workdir step3/${workdir}
```

```bash
qmap step3_md.py           # --dry-run first to read the script and the task list
```

Give the file `#!/usr/bin/env qmap` on line one and `chmod +x` it, and
`./step3_md.py` submits it directly.

The Python runs once, here, on the login node: it is what decides the list of
tasks. Nothing of it reaches the compute node.

- A **`!` at the top level runs immediately** while that happens, which is how
  `mkdir -p step3` exists before anything is submitted. `--dry-run` prints those
  without running them.
- A **`!` inside a loop is collected, not run** — one task per iteration of the
  innermost loop. Several `!` lines in one iteration become a single task joined
  with `&&`, so they stay in order on one node and stop at the first failure.
- **`${name}` and `$name` interpolate from the surrounding Python scope** and are
  shell-quoted, so a path with a space survives. A name that is not a Python
  variable is left alone — `${TMPDIR}` and `${SLURM_JOB_ID}` resolve on the
  compute node as usual.

Header keys map to the options below: `workload_manager`/`scheduler`, `account`,
`time`/`walltime`, `core`/`cores`, `cores_per_gpu`, `gpu`, `nodes`, `queue`/`partition`,
`mem`, `qos`, `constraint`, `concurrency`, `dependency`, `conda`, `module`,
`directive`, `export`, `setup`, `name`, `logdir`, `template`. A bare `#name` is a registered template:

```python
#Gautschi_H100_1GPU_4h
#account=siyoungk
#time=2:00:00
```

An empty value (`#account=`) is a blank left to fill in and is ignored. A
comment with a space after the `#` is prose. Anything else in the header is a
typo and stops the run. Flags after the file name go to `qmap` and override the
header, so `qmap job.py --dry-run --account other` works.

The job name defaults to the file's stem, and anything the Python raises stops
everything before a single task is submitted.

## Templates

A template is a set of submission options saved under a name — an account, a
queue, a core count, a walltime — so that a submission names a cluster setup
instead of respelling it.

**Which ones do I have?**

```bash
qmap templates
```

Each is listed with its options and its note:

```
Gautschi_H100_1GPU_4h   (shipped)
    --scheduler slurm --queue ai --nodes 1 --cores 14 --gpu 1 --walltime 4h
    Purdue Gautschi, ai partition, one H100. A Gautschi-H node holds
    8 x H100, 2 x Intel Xeon Platinum 8480+ and 112 CPU cores, so one
    GPU's share is 14 cores. Pass --account for your own allocation.
    needs: --account
```

**What exactly does one do?**

```bash
qmap show Gautschi_H100_1GPU_4h
```

which prints the note, the options, and the directives they turn into:

```
  options
    --scheduler slurm
    --queue ai
    --nodes 1
    --cores 14
    --gpu 1
    --walltime 4h

  directives
    #SBATCH -J Gautschi_H100_1GPU_4h
    #SBATCH -p ai
    #SBATCH -N 1
    #SBATCH -n 1
    #SBATCH -c 14
    #SBATCH --gres=gpu:1
    #SBATCH -t 4:00:00
    #SBATCH -o logs_Gautschi_H100_1GPU_4h/%A_%a.out
    #SBATCH -e logs_Gautschi_H100_1GPU_4h/%A_%a.err
    #SBATCH --array=1-N
```

`qmap directives NAME` prints that block on its own, ready to paste into a
hand-written script, and `qmap directives NAME lsf` prints the other
scheduler's.

**Registering one.** Either spell out the options:

```bash
qmap register Gautschi_H100_1GPU_4h --scheduler slurm --account my-account \
    --queue ai --nodes 1 --cores 14 --gpu 1 --walltime 4h \
    --note 'A node is 8 x H100 with 112 cores, so one GPU is 14 of them.'

qmap forget Gautschi_H100_1GPU_4h     # delete one
```

or read them out of a script you already have — see *Converting an existing
script* below.

A template's options are spliced in where `--template` appears, exactly as if
you had typed them there, so **anything after it wins**:

```bash
qmap --template Gautschi_H100_1GPU_4h --account other --cores 8 ...
```

Repeatable options (`--module`, `--directive`, `--setup`, `--export`)
accumulate; `--no-module` clears the modules a template would load. **An empty
value clears anything else a template set**, which is how you drop a
constraint rather than replace it:

```bash
qmap --template Lilac_A100_1GPU_4h --constraint '' ...   # any free GPU host
``` Templates
may reference other templates, so a personal one can build on a cluster one:

```bash
qmap register Gautschi_H100_1GPU_24h --template Gautschi_H100_1GPU_4h \
    --walltime 24h --concurrency 64 --conda ommflow
```

**What ships with qmap.** Four templates describing clusters this was written
against — `Gautschi_H100_1GPU_4h`, `Gautschi_4CPU_4h`, `Lilac_A100_1GPU_4h`,
`Lilac_A100_1GPU_168h`. They carry the hardware and queue, never an account,
because that part is yours. A template can say so:

```
# require: --account
```

and qmap then refuses to submit without it, naming what is missing, rather
than letting the scheduler reject the job a moment later:

```
$ qmap --template Gautschi_4CPU_4h --inputs 'data/*.pdb' -- analyse {input}
qmap: template Gautschi_4CPU_4h needs --account, which it cannot know for you.
```

Pass `--account` each time, or register your own copy that carries it —
`qmap register` writes to `~/.config/qmap/templates/`, which is searched first,
so your version of a name replaces the shipped one:

```bash
qmap register Gautschi_4CPU_4h --from - --require account <<'EOF'
#SBATCH -A my-allocation -p cpu -N 1 -c 4 -t 4:00:00
EOF
```

Templates are `*.args` files, one argument per line, so a value may contain
spaces without any escaping rules. Edit them by hand or re-register to
replace. `qmap register` checks the option names, so a typo is caught then
rather than at the next submission. Nothing is applied unless you name it:
there are no implicit defaults to inherit.

Name them for what you get rather than for the cluster alone — a walltime and
a GPU count belong in the name, since one cluster has as many useful shapes as
you have workloads:

```
Gautschi_H100_1GPU_4h
Lilac_A100_1GPU_168h
```

## One-liners: `qmap`

**The whole idea.** One task per file, and nothing in the script you did not ask for:

```bash
qmap --inputs 'raw/*.dcd' -- gzip -9 {input}
```

**A real MD run on Gautschi**, 10 ns per prepared structure, 64 tasks at a time:

```bash
qmap --template Gautschi_H100_1GPU_4h --account my-allocation \
     --inputs 'boltz_results_*/predictions/*/*.prepped.mae' \
     --name md --concurrency 64 \
     --conda ommflow --logdir logs_md \
     -- boonza md '{input}' \
        --workdir '{input.parent}/md_{input.stem}' \
        --production-ns 10 --platform CUDA --early-stop --monitor-chain L
```

**The same work on Lilac** — one word changes:

```bash
qmap --template Lilac_A100_1GPU_4h ...
```

**Continue where it left off.** Resubmit the same command; tasks whose marker
exists exit immediately:

```bash
qmap --template Gautschi_H100_1GPU_4h --account my-allocation \
     --inputs 'preds/*/*.mae' \
     --done-when '[ -f {input.parent}/md_{input.stem}/DONE ]' \
     -- boonza md '{input}' --workdir '{input.parent}/md_{input.stem}'
```

**A parameter sweep** rather than a list of files. The line is split on
whitespace and the fields are positional arguments:

```bash
printf '%s\n' 'sys/apo.pdb 300' 'sys/holo.pdb 310' 'sys/mut.pdb 300' > sweep.txt

qmap --list sweep.txt --name sweep --cores 8 --walltime 4:00:00 \
     -- python run.py --pdb '{0}' --temp '{1}' --tag '{0.stem}_T{1}'
```

**Numbered output directories**, and a payload that needs a shell:

```bash
qmap --inputs 'frames/*.pdb' --name score --cores 4 \
     -- sh -c 'mkdir -p out/{index:04d}; score {input} | awk "{{print \$2}}" > out/{index:04d}/{input.stem}.txt'
```

**From anything that produces a list:**

```bash
find . -name '*.yaml' -newer stamp | qmap --stdin --name predict --gpu 1 -- boltz predict {input}
```

Quote the placeholders so your own shell leaves them alone, and use `--dry-run`
to read the script before anything is submitted.

### Converting an existing script

`qmap convert` reads the directives out of a script you already have — either
`#SBATCH`/`#BSUB` lines or the flags on an `sbatch`/`bsub` command line — and
prints the other scheduler's:

```bash
qmap convert step3_md.sub              # to the other scheduler
qmap convert step3_md.sub slurm        # or name the target
pbpaste | qmap convert - slurm         # from a pasted block
```

It says what it cannot carry across rather than guessing: a value the shell
would have expanded (`-t $WALLTIME`), a flag with no counterpart (`--qos` on
LSF), and anything unrecognised is kept as a raw directive.

The same reading turns a pasted block straight into a template, minus the job
name, which belongs to the job rather than to the cluster:

```bash
qmap register Lilac_A100_1GPU_168h --from boltz/submissions/lsf/boltz_1GPU.sub \
    --note 'gpuqueue, one A100, 12 cores, 7 days'
qmap directives Lilac_A100_1GPU_168h slurm     # read it back as either scheduler
```

### Walltime

The two schedulers spell durations differently and neither rejects the other's
spelling — LSF reads `24:00` as 24 hours, Slurm reads it as 24 minutes. Write
it with units and qmap renders the right form for whichever scheduler:

```bash
--walltime 4h        # -> #SBATCH -t 4:00:00   or   #BSUB -W 4:00
--walltime 90m       # -> 1:30:00              or   1:30
--walltime 2d12h     # -> 60:00:00             or   60:00
```

A bare `24:00` is still passed through exactly as typed, and converting a
template that carries one warns rather than reinterpreting it.

### Cores, tasks and workers

Say you want 64 CPUs' worth of work, as 16 workers of 4 CPUs each. There are
two different jobs hiding in that sentence.

**16 array tasks at a time, 4 cores each.** The workers are independent, each
handling one input, and the scheduler starts them wherever there is room:

```bash
qmap --cores 4 --concurrency 16 --inputs 'data/*.pdb' -- analyse {input}
```

Nothing asks for 64 cores; 16 × 4 are simply in flight at once. This is what a
job array is for, it starts as soon as any 4 cores are free, and a worker that
dies takes one input down with it rather than the whole job.

**One allocation of 64, divided 16 × 4.** Right when the workers must share a
node — MPI ranks, or a pool whose parts talk to each other:

```bash
qmap --tasks 16 --cores 4 --nodes 1 --inputs 'data/*.pdb' -- srun analyse {input}
```

```
#SBATCH -N 1        #BSUB -n 64
#SBATCH -n 16       #BSUB -R "span[hosts=1]"
#SBATCH -c 4
```

qmap still runs your command once per array element, so `--tasks` only means
anything if the command itself starts the workers — `srun`, `mpirun`, a
`multiprocessing.Pool`, `xargs -P`. Without one of those, 15 of the 16 sit
idle. The simpler version of this is `--cores 64` with a payload that spawns
its own 16 workers.

### Partition vs QOS

`--queue` is the partition (Slurm `-p`, LSF `-q`). Slurm's `-q` is a separate
thing, the QOS, and has its own flag:

```bash
qmap --template Gautschi_H100_1GPU_4h --account my-allocation --qos normal ...
```

There are deliberately no single-letter aliases: `-q`, `-p`, `-c`, `-n` and `-A`
mean different things to LSF and Slurm, so qmap refuses them rather than
guessing. Anything with no flag at all goes through `--directive`.

### Flag table

What each qmap option becomes. Nothing is emitted unless you set it, except
the four rows marked *always*.

| qmap | job-file header | Slurm | LSF |
| --- | --- | --- | --- |
| `--name NAME` | `#name=` | `-J NAME` *(always)* | part of `-J "NAME[1-N]"` *(always)* |
| `--account A` | `#account=` | `-A A` | `-P A` |
| `--queue Q` | `#queue=`, `#partition=` | `-p Q` | `-q Q` |
| `--qos Q` | `#qos=` | `-q Q` | — (refused; use `--directive`) |
| `--nodes N` | `#nodes=` | `-N N` | — (implied by `span[hosts=1]`) |
| `--cores N` | `#core=`, `#cores=` | `-n 1` + `-c N` | `-n N`, plus `-R "span[hosts=1]"` when N > 1 |
| `--tasks N` | `#tasks=` | `-n N` | `-n N×cores` (LSF counts slots) |
| `--cores-per-gpu N` | `#cores_per_gpu=` | derives `--cores` from `--gpu`, nothing of its own | same |
| `--gpu N` | `#gpu=` | `--gres=gpu:N` | `-gpu "num=N:mode=exclusive_process"` |
| `--mem SPEC` | `#mem=` | `--mem=SPEC` | `-R "rusage[mem=SPEC]"` |
| `--constraint C` | `#constraint=` | `-C C` | `-R C` |
| `--walltime T` | `#time=`, `#walltime=` | `-t [d-]hh:mm:ss` | `-W [hh]:mm` |
| `--concurrency N` | `#concurrency=` | `%N` on `--array` | `%N` on `-J` |
| `--dependency ok:ID` | `#dependency=` | `-d afterok:ID` | `-w "done(ID)"` |
| `--dependency any:ID` | `#dependency=` | `-d afterany:ID` | `-w "ended(ID)"` |
| `--dependency fail:ID` | `#dependency=` | `-d afternotok:ID` | `-w "exit(ID)"` |
| `--dependency start:ID` | `#dependency=` | `-d after:ID` | `-w "started(ID)"` |
| `--dependency singleton` | `#dependency=` | `-d singleton` | — (refused; use `--directive`) |
| `--logdir DIR` | `#logdir=` | `-o DIR/%A_%a.out`, `-e …err` *(always)* | `-o DIR/%J_%I.out`, `-e …err` *(always)* |
| *(the input count)* | — | `--array=1-N` *(always)* | `[1-N]` on `-J` *(always)* |
| `--directive TEXT` | `#directive=` | `#SBATCH TEXT` | `#BSUB TEXT` |
| — | — | — | `-L /bin/bash` *(always)* |

Durations are the one place the two disagree silently, so `--walltime` also
takes units — `4h`, `90m`, `2d12h` — and renders the right spelling for each.

These options are qmap's own and produce no directive; they shape the script
body or the run instead:

| qmap | job-file header | what it does |
| --- | --- | --- |
| `--template NAME` | `#NAME` on its own line | splices in a registered set of options |
| `--conda ENV` | `#conda=`, `#env=` | sources conda's `profile.d`, then `conda activate ENV` |
| `--module M` | `#module=` | `module load M`, repeatable |
| `--no-module` | — | drops modules a template would load |
| `--export VAR=VAL` | `#export=` | `export VAR=VAL` in the task |
| `--setup 'LINE'` | `#setup=` | a shell line before the command |
| `--chdir DIR` | — | the directory tasks run in (default: where you submit) |
| `--inputs`, `--list`, `--input`, `--stdin` | — | what to map the command over |
| `--done-when 'TEST'` | — | skip inputs already finished |
| `--scheduler S` | `#workload_manager=`, `#scheduler=` | `lsf`, `slurm`, `local`, or auto from `$PATH` |
| `--dry-run` | — | print the script, submit nothing |
| `--parsable` | — | print the new job id alone on stdout |

## Running locally: `--scheduler local`

The same job, with no scheduler anywhere: `--scheduler local` runs the tasks
here and now instead of writing directives for a cluster.

```bash
qmap --scheduler local --concurrency 4 \
     --inputs 'raw/*.dcd' -- gzip -9 {input}
```

It is the same array, the same `job.sh` and the same `inputs.txt` — only the
directive block is gone and qmap walks the task indices itself. That makes it
the honest way to try a job on a handful of inputs before sending ten thousand
of them to a queue: change one word and the command line stays put.

**`--concurrency` is the parallelism, and it defaults to one.** Everywhere else
an omitted `--concurrency` means "no cap", which locally would mean starting
every task at once; here it means one at a time until you say otherwise.

**Nothing is allocated.** A local run has no one to ask for an account, a
queue, a walltime, a GPU or memory, so those are ignored and qmap says which
ones it dropped. That includes the GPU: two tasks running at once share
whichever one the machine has. Templates keep working — `--template
Gautschi_H100_1GPU_4h --scheduler local` runs the same command line here and
warns about the rest.

**Each task gets its own pair of log files**, as it would under a scheduler,
so parallel tasks do not interleave into one unreadable stream:

```
logs_<name>/local-<stamp>_<index>.out
logs_<name>/local-<stamp>_<index>.err
```

Failures are counted rather than fatal: the run finishes the remaining tasks,
lists the ones that came back non-zero, and exits non-zero itself.

```
  [1/3] ok
  [2/3] FAILED (exit 1) logs_demo/local-20260919-173404_2.err
  [3/3] ok
qmap: 1 task(s) failed: 2
qmap: rerun those with: .qmap/demo-20260919-173404/run.sh 2
```

That `run.sh` is an ordinary script kept beside `job.sh`. Run it again to
repeat the whole array, or name task numbers to redo only those. To run one
task in the foreground with its output on your terminal, skip the driver
entirely:

```bash
QMAP_TASK_ID=2 .qmap/demo-20260919-173404/job.sh
```

`auto` never chooses `local`. Finding no `sbatch` and no `bsub` is the normal
state of a laptop, and it should not quietly mean "run a thousand tasks here";
ask for it by name.

## Chaining jobs: `--dependency`

A job that has to wait for another one is spelled differently by each
scheduler, so `--dependency` takes one neutral spelling and renders whichever
is needed. A bare id means `ok:`, which is the common case:

```bash
qmap step3_md.py --dependency 12345        # after 12345 succeeds
qmap step3_md.py --dependency any:12345    # after it ends, however it ended
```

| qmap | means | Slurm | LSF |
| --- | --- | --- | --- |
| `ok:ID` | after ID succeeded | `afterok:ID` | `done(ID)` |
| `any:ID` | after ID ended, any status | `afterany:ID` | `ended(ID)` |
| `fail:ID` | after ID failed | `afternotok:ID` | `exit(ID)` |
| `start:ID` | after ID started | `after:ID` | `started(ID)` |
| `singleton` | after every earlier job of yours with this `--name` | `singleton` | — |

Repeat the flag to wait on several; they are ANDed. Only a plain numeric job
id renders for both schedulers, so an array element, a job name or an `or`
is refused rather than half-translated — write those with `--directive`.

### Running the same job N times, one after another

Chaining past a walltime limit wants `any:`, not `ok:` — a link that times
out or dies has still left the queue, and `ok:` would wedge the rest of the
chain behind it.

On Slurm this needs no job ids at all. `singleton` means "wait for every
earlier job of mine with this name", so ten submissions sharing one `--name`
serialise themselves:

```bash
for i in $(seq 1 10); do
    qmap step3_md.py --name md_chain --dependency singleton
done
```

The chain is keyed on name and user, so pick a name you are not reusing
elsewhere.

On either scheduler, `--parsable` prints the new job id and nothing else,
which is what makes an explicit chain short:

```bash
prev=
for i in $(seq 1 10); do
    dep=(); [ -n "$prev" ] && dep=(--dependency "any:$prev")
    prev=$(qmap step3_md.py --name "md_$i" "${dep[@]}" --parsable)
    echo "link $i is job $prev"
done
```

Each link is an array job, and a dependency on it waits for the whole array
to drain, not just its first task.

Ten links only help if links 2 to 10 pick up where the last one stopped, so
pair this with `--done-when` (or your tool's own checkpoint/restart), or you
will run the same work ten times:

```
#done_when=test -s md_${pdb}/production.done
```

Without `--parsable` the job id is printed with the rest of the summary and
written to `.qmap/<name>-<stamp>/jobid`.

## What a submission leaves behind

```
.qmap/<name>-<timestamp>/
    job.sh        the generated script, exactly as submitted
    inputs.txt    the input list that script indexes into
    jobid         the id the scheduler gave it, once submitted
    run.sh        --scheduler local only: the driver that walks the tasks
    failed        --scheduler local only: the tasks that came back non-zero
logs_<name>/      per-task stdout and stderr
```

`job.sh` stands alone: it reads `QMAP_TASK_ID`, `SLURM_ARRAY_TASK_ID` or
`LSB_JOBINDEX`, whichever is set, so when a job needs something no flag covers,
edit that file and resubmit it by hand with `sbatch` or `bsub <` — or run one
task right here with `QMAP_TASK_ID=7 .qmap/<name>-<stamp>/job.sh`. `--dry-run` prints it
without submitting, and writes it to `.qmap/dry-run/`, one directory reused
by every preview, so previewing leaves no trail of stamped directories or log
directories behind.

Each submission gets its own directory on purpose. A running array reads its
list lazily, one task at a time, so a single shared list file would let a
resubmission rewrite a live job's inputs underneath it.

Add `.qmap/` and `logs_*/` to the project's `.gitignore`.

## Templating reference

Python's `str.format` over `pathlib`, so these mean what they would in Python.
For `foo/bar/x.prepped.mae`:

| placeholder | value |
| --- | --- |
| `{input}` or `{}` | `foo/bar/x.prepped.mae` |
| `{input.name}` | `x.prepped.mae` |
| `{input.stem}` | `x.prepped` |
| `{input.suffix}` | `.mae` |
| `{input.parent}` | `foo/bar` |
| `{input.parent.name}` | `bar` |
| `{input.parent.parent}` | `foo` |
| `{index}`, `{i}` | 1-based task index; `{index:04d}` zero-pads |
| `{n}` | total number of tasks |
| `{0}`, `{1}`, `{1.stem}` | line split on whitespace, as positional args |
| `{{`, `}}` | a literal brace, so `awk '{{print $1}}'` works |
| `${VAR}` | passed through, resolved in the task's environment |

Everything else: `qmap --help`.

## License

MIT.
