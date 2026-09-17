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

**Nothing is set for you.** An account, queue, walltime or core count appears
in the generated script only because you asked for it — on the command line,
or through a template you registered. There is no implicit cluster default.

## Install

```bash
git clone https://github.com/siyoungkimlab/qmap.git ~/qmap
export PATH="$HOME/qmap/bin:$PATH"
```

`git pull` in `~/qmap` updates it. Symlinking works as well —
`ln -s ~/qmap/bin/qmap ~/.local/bin/qmap` — since qmap follows symlinks to
find its own files.

```
bin/qmap              the scheduler side, and the only thing on PATH
libexec/qmap-jobfile  runs a Python job file
libexec/qmap-directives   reads an existing #SBATCH/#BSUB block
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
- `task('...')` and `sh('...')` are available when you would rather be explicit
  than rely on where the `!` sits.

Header keys map to the options below: `workload_manager`/`scheduler`, `account`,
`time`/`walltime`, `core`/`cores`, `cores_per_gpu`, `gpu`, `nodes`, `queue`/`partition`,
`mem`, `qos`, `constraint`, `concurrency`, `conda`, `module`, `directive`, `export`,
`setup`, `name`, `logdir`, `template`. A bare `#name` is a registered template:

```python
#Gautschi_H100_1GPU_4h
#time=2:00:00
#gpu=1
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
Gautschi_H100_1GPU_4h
    --scheduler slurm --account siyoungk --queue ai --nodes 1 --cores 14 --gpu 1 --walltime 4h
    Each Gautschi-H node has 8 x H100 GPUs and 112 CPU cores, so one GPU's share is 14.
```

**What exactly does one do?**

```bash
qmap show Gautschi_H100_1GPU_4h
```

which prints the note, the options, and the directives they turn into:

```
  options
    --account siyoungk
    --queue ai
    --cores 14
    --walltime 4h

  directives
    #SBATCH -A siyoungk
    #SBATCH -p ai
    #SBATCH -c 14
    #SBATCH -t 4:00:00
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
accumulate; `--no-module` clears the modules a template would load. Templates
may reference other templates, so a personal one can build on a cluster one:

```bash
qmap register Gautschi_H100_1GPU_24h --template Gautschi_H100_1GPU_4h \
    --walltime 24h --concurrency 64 --conda ommflow
```

Templates live in `~/.config/qmap/templates/*.args`, one argument per line, so
a value may contain spaces without any escaping rules. Edit them by hand or
re-register to replace. `qmap register` checks the option names, so a typo is
caught then rather than at the next submission. None ships with qmap: a
template applies only when you name it, and there are no implicit defaults to
inherit.

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
qmap --template Gautschi_H100_1GPU_4h \
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
qmap --template Gautschi_H100_1GPU_4h --inputs 'preds/*/*.mae' \
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
qmap --template Gautschi_H100_1GPU_4h --qos normal ...
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
| `--scheduler S` | `#workload_manager=`, `#scheduler=` | `lsf`, `slurm`, or auto from `$PATH` |
| `--dry-run` | — | print the script, submit nothing |

## What a submission leaves behind

```
.qmap/<name>-<timestamp>/
    job.sh        the generated script, exactly as submitted
    inputs.txt    the input list that script indexes into
logs_<name>/      per-task stdout and stderr
```

`job.sh` stands alone: it reads `SLURM_ARRAY_TASK_ID` or `LSB_JOBINDEX`,
whichever is set, so when a job needs something no flag covers, edit that file
and resubmit it by hand with `sbatch` or `bsub <`. `--dry-run` prints it
without submitting.

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
