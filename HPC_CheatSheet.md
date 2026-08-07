# HPC Command-Line Cheat Sheet

Job monitoring, file transfer, text processing, and other essentials for working on a PBS Professional cluster (Avalon / QIMR Berghofer). Commands verified against real usage during an active graphREML/graphld HPC session.

## Contents

1. [Submitting and Checking Jobs](#1-submitting-and-checking-jobs)
2. [Detailed Job Monitoring](#2-detailed-job-monitoring)
3. [Controlling Jobs](#3-controlling-jobs)
4. [File Transfer (scp, rsync)](#4-file-transfer-scp-rsync)
5. [grep / egrep — Searching Text and Logs](#5-grep--egrep--searching-text-and-logs)
6. [sed — Editing Text Streams](#6-sed--editing-text-streams)
7. [awk — Column-Based Text Processing](#7-awk--column-based-text-processing)
8. [Environment Modules](#8-environment-modules)
9. [Disk Usage and Quotas](#9-disk-usage-and-quotas)
10. [Persistent Sessions (tmux)](#10-persistent-sessions-tmux)
11. [Process Management](#11-process-management)
12. [Archiving and Compression](#12-archiving-and-compression)
13. [PBS Environment Variables and Troubleshooting](#13-pbs-environment-variables-and-troubleshooting)
14. [Practical Combined Recipes](#14-practical-combined-recipes)
15. [Git Quick Reference](#15-git-quick-reference)
16. [Bash Productivity Tips](#16-bash-productivity-tips)
17. [PBS Job Chaining and Dependencies](#17-pbs-job-chaining-and-dependencies)

---

## 1. Submitting and Checking Jobs

### `qsub` — submit a job

```bash
qsub myscript.pbs                                        # submit a script; resources come from #PBS lines inside it
qsub -l ncpus=1,mem=22gb,walltime=2:00:00 myscript.pbs    # override/add resources from the command line
qsub -q bigmem myscript.pbs                               # submit to a specific queue
qsub -I -l mem=1gb,walltime=8:00:00                       # interactive session instead of a batch script
```

**Explanation:** `qsub` hands your script to the PBS server, which caches a copy of it at that exact moment — editing the file afterward will **not** change what the already-submitted job runs. Command-line `-l` options take precedence over `#PBS -l` lines inside the script if both are present.

### `qstat` — check job/queue status

| Command | What it shows |
|---|---|
| `qstat -u $USER` | All of your jobs, across all states (queued, running, held) |
| `qstat -u $USER -1n` | Same, but one line per job with the execution node(s) appended |
| `qstat -f <jobid>` | Full detail for one job: resources requested/used, paths, queue, state |
| `qstat -fx <jobid>` | Same as `-f`, but also works for jobs that have already finished |
| `qstat -x <jobid>` | Short summary for a finished job |
| `qstat -H <jobid>` | Historical/queue-style summary line for a finished job |
| `qstat -Q` | List all queues and their current load |
| `qstat -Qf <queue>` | Full detail for one queue: walltime caps, ACLs, enabled/started status |

---

## 2. Detailed Job Monitoring

### The one-liner: real usage vs. what you requested

```bash
qselect -u $USER -s R | xargs qstat -f | egrep "^Job|resources_used.cpupercent|resources_used.mem|resources_used.walltime|exec_vnode"
```

- `qselect -u $USER -s R` : lists job IDs for your **R**unning jobs (`-s Q` = Queued, `-s H` = Held)
- `| xargs qstat -f` : pipes each job ID into `qstat -f` for its full attribute block
- `| egrep "..."` : filters down to just CPU%, memory, walltime used, and execution node

Comparing `resources_used.mem` against what you requested tells you if you're over-requesting (wasting shared capacity) or under-requesting (risking an OOM kill).

### `jobperf` : live resource dashboard

```bash
module load jobperf/0.1.0
jobperf -http -http-disable-auth -w <jobid>     # NOTE: no ".hpcpbs02" suffix on the job ID
```

Starts a small web server, prints a URL with live CPU/memory/GPU graphs. `Ctrl+C` stops watching ; does not kill the job.

For long jobs, have it self-monitor and save to a database:

```bash
# Add near the top of your PBS script, after module loads:
module load jobperf/0.1.0
jobperf -record -w -rate 30s -http-disable-auth -http &
```

The trailing `&` backgrounds it so your actual commands still run in the foreground. Use a longer `-rate` (e.g. `30s`) for multi-day jobs to avoid an oversized stats database.

**One-time setup required** (do this once, ever):

```bash
ssh-keygen                              # hit Enter through all prompts (no passphrase)
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

> **Caution:** never copy the generated `~/.ssh/id_rsa` file off the HPC ; it grants passwordless access to your account.

---

## 3. Controlling Jobs

| Command | Purpose | Example |
|---|---|---|
| `qalter` | Change attributes of a queued/running job | `qalter -l walltime=72:00:00 <jobid>` |
| `qdel` | Cancel/kill a job | `qdel <jobid>` |
| `qhold` | Pause a queued job so it won't start | `qhold <jobid>` |
| `qrls` | Release a held job | `qrls <jobid>` |
| `qsig -s suspend` | Pause a **running** job (process suspended, not killed) | `qsig -s suspend <jobid>` |
| `qsig -s resume` | Resume a suspended job | `qsig -s resume <jobid>` |

> **Careful with `qalter walltime`:** double-check the `HH:MM:SS` digits — get it wrong and you can accidentally *shorten* a job's walltime, killing it immediately instead of extending it.

### Selecting jobs by criteria

```bash
qselect -u $USER | xargs qdel                                    # kill all of your jobs
qselect -ts.gt.01210900 -ts.lt.01211000 -u $USER | xargs qdel    # kill jobs started in a specific time window
```

---

## 4. File Transfer (scp, rsync)

### scp — secure copy

```bash
scp <source> <destination>
```

Either side can be local (plain path) or remote (`user@host:path`).

```bash
# Local -> cluster
scp myfile.txt user@admin.ad.lan:/working/filelocation/

# Cluster -> local (run FROM your local machine)
scp user@admin.ad.lan:/working/filelocation/.../local/path .

# Whole directory, recursively
scp -r user@admin.ad.lan:/working/filelocation/.../local/path/ .
```

| Flag | Meaning |
|---|---|
| `-r` | Recursive — copy a whole directory |
| `-P <port>` | Non-default SSH port (capital P) |
| `-p` | Preserve timestamps/permissions (lowercase p) |
| `-C` | Compress during transfer — helps with large `.sumstats`/`.tsv` files |
| `-i <keyfile>` | Use a specific private key |

> **The trailing dot:** in `scp user@host:/path/to/file .`, the `.` means "copy into my current directory, keeping the original filename." Forgetting it is a common source of errors.

### rsync — smarter, resumable copying

Preferred over `scp` for large files or repeated syncs, since it only transfers what's changed and can resume interrupted transfers.

```bash
rsync -avz user@admin.ad.lan:/working/filelocation/.../runs/ ./local_runs/
rsync -avzP user@admin.ad.lan:/working/filelocation/working/.../local_file .   # -P shows progress + allows resume
```

- `-a` = archive mode (preserves permissions, timestamps, symlinks, recurses into directories)
- `-v` = verbose
- `-z` = compress during transfer
- `-P` = show progress bar and allow resuming a partial transfer

---

## 5. grep / egrep — Searching Text and Logs

`grep` searches text for lines matching a pattern. `egrep` (= `grep -E`) enables extended regex — `|`, `+`, `?`, `{}` work without backslash-escaping.

```bash
grep "ERROR" graphreml.err          # lines containing "ERROR"
grep -i "error" graphreml.err       # case-insensitive
grep -v "INFO" graphreml.out        # invert match: lines that DON'T contain "INFO"
grep -c "WARNING" graphreml.out     # count matches instead of printing them
grep -n "iteration" graphreml.out   # show line numbers
```

### Multiple patterns (OR)

```bash
egrep "^Job|resources_used.cpupercent|resources_used.mem" file.txt
```

- `|` means OR — match any line containing any listed pattern
- `^Job` = "Job" at line-start only (`^` anchors to line start)

### Context lines

```bash
grep -A5 "Traceback" graphreml.err     # match + 5 lines After
grep -B3 "ValueError" graphreml.err    # match + 3 lines Before
grep -B2 -A5 "ERROR" graphreml.err     # both
grep -m1 "Error" graphreml.err         # stop after the FIRST match
```

### Recursive / filename search

```bash
grep -rl "population" data/            # -r recurse, -l print only matching FILENAMES
find . -iname "*chr3_11805_1281684*"   # find = filenames; grep = file CONTENTS
```

---

## 6. sed — Editing Text Streams

`sed` reads text line by line and applies an edit — most commonly find-and-replace.

### Basic substitution: `s/pattern/replacement/`

```bash
sed 's/UKBB/EUR/' file.txt          # replace FIRST match per line, print to screen
sed 's/UKBB/EUR/g' file.txt         # g = global -- replace ALL matches per line
sed -i 's/UKBB/EUR/g' file.txt      # -i = in-place -- actually MODIFY the file
sed -i.bak 's/UKBB/EUR/g' file.txt  # keep a backup as file.txt.bak first (safer)
```

> **Always test before `-i`:** run without `-i` first and eyeball the output. Once you add `-i`, original content is gone unless you also used a `.bak` suffix.

### A real example, annotated

```bash
newname=$(echo "$file" | sed -E 's/ukbb\.EUR\.(1kg_chr[^.]+)\.path_distance=[0-9.]+\.l1_pen=[0-9.]+\.maf=[0-9.]+\.ALL\.edgelist/\1.UKBB.edgelist/')
```

- `-E` enables extended regex — `(` `)` and `+` work without backslashes
- `(1kg_chr[^.]+)` is a **capture group** — saves the matched text for reuse
- `[0-9.]+` matches one-or-more digits/dots — used for numeric parameter values
- `\1` in the replacement refers back to the first `(...)` group

### Other common operations

```bash
sed -n '5,10p' file.txt              # -n suppresses default output; print ONLY lines 5-10
sed '/^#/d' file.txt                 # delete lines starting with #
sed 's/^/PREFIX_/' file.txt          # prepend to every line
sed 's/$/_SUFFIX/' file.txt          # append to every line
sed 's/[[:space:]]\+/ /g' file.txt   # collapse multiple spaces/tabs into one
```

---

## 7. awk — Column-Based Text Processing

`awk` is built for tabular/columnar data (TSVs, CSVs, log files with consistent fields) — complements `grep`/`sed`, which are line-oriented.

```bash
awk '{print $1}' file.txt                  # print the 1st whitespace-separated column
awk -F',' '{print $6}' metadata.csv         # -F sets the field separator (comma here)
awk -F',' '$6=="EUR" {print $4}' metadata.csv   # print column 4 only where column 6 equals "EUR"
awk '{sum+=$3} END {print sum}' file.txt    # sum column 3 across all lines
awk 'NR==1' file.txt                        # print only the first line (e.g. a header)
awk 'NR>1' file.txt                         # print everything EXCEPT the first line
```

**Example — count variants per chromosome in an annotation file:**

```bash
awk -F'\t' '{print $1}' baselineLD.annot | sort | uniq -c
```

---

## 8. Environment Modules

```bash
module avail                    # list ALL available software modules
module avail gpu                # list only modules under the "gpu" namespace
module load suitesparse/7.11.0  # load a specific module/version
module list                     # show what's currently loaded in this session
module unload suitesparse       # remove a loaded module
module purge                    # remove ALL loaded modules
```

**Explanation:** modules set environment variables (`PATH`, `LD_LIBRARY_PATH`, etc.) so a specific software version becomes available without conflicting with other versions installed on the system. Always `module load` inside your PBS script itself — don't rely on modules loaded in your interactive shell, since batch jobs start with a near-empty environment.

---

## 9. Disk Usage and Quotas

```bash
du -sh data/ldgms/                  # total size of one directory, human-readable
du -sh */ 2>/dev/null | sort -rh    # size of each subdirectory, largest first
df -h /working/lab_tracyo           # free space on the filesystem containing this path
quota -s                            # your personal quota usage, if configured
```

**Explanation:** `du` (disk usage) measures actual file sizes; `df` (disk free) measures filesystem-level free space, which can differ from what `du` suggests if there's a quota involved. Check both when a job fails mysteriously partway through a large download or write.

---

## 10. Persistent Sessions (tmux)

Keeps a session alive on the login node even if your SSH connection drops — useful for long-running interactive monitoring (e.g. watching `jobperf` or `tail -f` for hours).

```bash
tmux new -s graphld           # start a new named session
tmux attach -t graphld        # reattach to an existing session after disconnecting
tmux ls                       # list all your active sessions
# Inside tmux: Ctrl+b then d   -- detach (session keeps running in the background)
# Inside tmux: Ctrl+b then c   -- open a new window within the session
```

> **Note:** don't use `tmux` to run actual compute-heavy work on the login node itself — it's still just for keeping a *monitoring or editing* session alive, not a substitute for submitting a proper PBS job.

---

## 11. Process Management

```bash
ps aux | grep graphld           # find running processes matching "graphld"
ps -u $USER                     # all of your own processes
top                             # live process/resource viewer (press q to quit)
htop                            # nicer interactive version of top, if installed
kill <pid>                      # politely ask a process to terminate
kill -9 <pid>                   # force-kill (last resort)
```

**Explanation:** on the login node, `ps`/`kill` let you check whether a long-running command (like a `make download_precision` you started) is still active before assuming it's stalled or finished.

---

## 12. Archiving and Compression

```bash
tar -xzf archive.tar.gz -C target_dir/      # extract a .tar.gz into target_dir/
tar -czf archive.tar.gz mydir/              # create a .tar.gz from mydir/
curl -L -o - <url> | tar -xz -C target_dir/ # stream-download and extract in one step (used for Zenodo downloads)
gunzip file.gz                              # decompress a .gz file in place
zcat file.gz | head                         # view the start of a .gz file without fully decompressing it
```

The `curl | tar` pattern (piping a download directly into extraction without saving the archive to disk first) is exactly what `make download_precision` used for the Zenodo LDGM downloads earlier — saves disk space since the compressed archive is never written to disk.

---

## 13. PBS Environment Variables and Troubleshooting

### Useful variables available inside a running job

| Variable | Meaning |
|---|---|
| `$PBS_JOBID` | The full job ID (e.g. `34622577.hpcpbs02`) |
| `$PBS_O_WORKDIR` | The directory you were in when you ran `qsub` |
| `$PBS_ARRAY_INDEX` | Current index, only set inside a job array (`#PBS -J`) |
| `$PBS_NODEFILE` | Path to a file listing allocated nodes, for MPI jobs |

### Passing environment variables into a job

```bash
export FOO=bar
qsub -v FOO myscript.pbs        # pass just $FOO through
qsub -V myscript.pbs            # pass your ENTIRE current environment through
```

> Only works for variables referenced in the script body — **not** for `#PBS` directives at the top of the script.

### Debugging a job that failed mysteriously

```bash
tracejob <jobid>                 # detailed scheduler-level history of a job (run on execution node)

# While a job is still running, live logs are on the execution node itself, BEFORE
# being copied back to your Output_Path/Error_Path on completion:
cat /var/spool/PBS/spool/<jobid>.OU     # live stdout
cat /var/spool/PBS/spool/<jobid>.ER     # live stderr
```

---

## 14. Practical Combined Recipes

**Watch a job's log live while checking resource usage periodically:**
```bash
tail -f runs/ecac2018_run1/graphreml.out &
watch -n 60 'qselect -u $USER -s R | xargs qstat -f | egrep "^Job|resources_used"'
```

**Find the first error in a crashed job's log, with context, and count total errors:**
```bash
grep -B2 -A10 -m1 "Error\|Traceback" runs/ecac2018_run1/graphreml.err
grep -c "Error" runs/ecac2018_run1/graphreml.err
```

**Batch-rename files matching a broken pattern (the `rename_ukbb_files.sh` pattern):**
```bash
for file in *; do
  newname=$(echo "$file" | sed -E 's/OLD_PATTERN/NEW_PATTERN/')
  if [ "$newname" != "$file" ]; then
    mv "$file" "$newname"
  fi
done
```
The `if [ "$newname" != "$file" ]` guard skips files that didn't match, so unrelated files are left untouched.

**Count files per category across a large directory:**
```bash
for pop in AFR AMR EAS EUR SAS UKBB; do
  n=$(ls data/ldgms/*."$pop".edgelist 2>/dev/null | wc -l)
  echo "$pop: $n files"
done
```

**Extract unique values from one column of a CSV:**
```bash
cut -d',' -f6 data/ldgms/metadata.csv | sort -u
```

---

## 15. Git Quick Reference

Useful if this repo (or your analysis scripts) live under version control alongside this cheat sheet.

```bash
git status                          # what's changed since the last commit
git diff                            # exact line-by-line changes, unstaged
git add file.pbs                    # stage a specific file
git add -A                          # stage everything changed
git commit -m "message"             # commit staged changes
git log --oneline -10                # last 10 commits, one line each
git describe --tags                  # how many commits ahead of the last tag you are
git pull                            # fetch + merge latest changes from remote
git push                            # push your commits to remote
git checkout -b my-branch           # create and switch to a new branch
```

**Checking exactly which commit a bug report applies to** (useful when reporting issues against a fast-moving repo like `graphld`):
```bash
git log -1 --format="%H %ci"        # exact commit hash + date
git describe --tags                 # e.g. v1.2.1-14-g6d7a2e9 = 14 commits past tag v1.2.1
```

---

## 16. Bash Productivity Tips

Small habits that save real time once they're muscle memory.

### History and repetition

```bash
!!                    # re-run the last command (e.g. "sudo !!" after a permission-denied error)
!$                    # the LAST ARGUMENT of the previous command -- e.g. "mkdir proj && cd !$"
!<n>                  # re-run history entry number <n> (see numbers with `history`)
Ctrl+R                # reverse-search history -- start typing, it finds matching past commands
history | grep "qsub" # search your full command history for anything containing "qsub"
```

### Navigation shortcuts

```bash
cd -              # jump back to the PREVIOUS directory you were in
pushd /some/path  # go to /some/path, but remember where you came from
popd              # jump back to wherever the last `pushd` was called from
Ctrl+A / Ctrl+E    # jump to the start / end of the current command line
Ctrl+W             # delete the word before the cursor
Ctrl+U             # delete from cursor to the start of the line
Ctrl+L             # clear the screen (like `clear`, but faster)
```

### Job control inside a shell (not PBS jobs -- your terminal's own background processes)

```bash
long_command &     # run in the background immediately
Ctrl+Z             # suspend the current foreground process
bg                 # resume the most recently suspended process, in the background
fg                 # bring the most recent background/suspended process to the foreground
jobs               # list background/suspended jobs in this shell session
```

### Aliases (add to `~/.bashrc`, then `source ~/.bashrc` to reload)

```bash
alias ll='ls -lh'
alias myq='qstat -u $USER -1n'
alias gs='git status'
```

> **Tip from this session's own troubleshooting:** an alias like `myq` above turns a command you'll type dozens of times a day into three keystrokes -- worth setting up early rather than retyping `qstat -u tsheringD -1n` every time.

### Brace expansion (avoid retyping shared prefixes/suffixes)

```bash
touch README.txt requirements.txt TODO.txt      # the verbose way
touch {README,requirements,TODO}.txt              # brace expansion -- same result, .txt typed once
mkdir project && cd project                       # the verbose way
mkdir project && cd $_                             # $_ = last argument of the previous command
```

---

## 17. PBS Job Chaining and Dependencies

For a multi-stage pipeline (e.g. `download -> fit graphREML -> run score test`) where each stage must wait for the previous one to genuinely succeed, don't poll `qstat` manually and submit the next stage by hand. Use PBS's built-in dependency mechanism instead.

### Basic chain: run job2 only after job1 finishes successfully

```bash
JOB1=$(qsub 01_download.pbs)
JOB2=$(qsub -W depend=afterok:$JOB1 02_run_reml.pbs)
JOB3=$(qsub -W depend=afterok:$JOB2 03_score_test.pbs)
```

`$JOB1` captures the job ID that `qsub` prints, and `-W depend=afterok:$JOB1` tells PBS "hold this job and don't run it until `$JOB1` has exited with status 0 (success)." Each subsequent job sits in the queue in a "Not Queued" hold state until its dependency clears.

### Common dependency types

| Clause | Meaning |
|---|---|
| `afterok:<id>` | Run only if `<id>` completed **successfully** (exit code 0) |
| `afternotok:<id>` | Run only if `<id>` **failed** (non-zero exit code) |
| `afterany:<id>` | Run after `<id>` finishes, **regardless** of success or failure |
| `before:<id>` | The reverse direction -- `<id>` won't start until THIS job starts |

> **Caution:** with `afterok`, if the upstream job fails, the downstream job is deleted due to "dependency not met" rather than sitting and waiting — check on it, don't assume silence means it's still queued.

### Why this beats a self-submitting ("recursive") script

A tempting shortcut is having a script `qsub` its own successor at the very end of its own execution. This is explicitly discouraged: if the job hits its walltime limit and gets killed by the scheduler, the final `qsub` line for the next stage never runs, and the whole chain silently stops with no error you'd notice until you go looking. `-W depend=` avoids this failure mode because all jobs are submitted up front, before any of them can be killed mid-script.

### Practical example: chaining this session's actual graphREML pipeline

```bash
#!/bin/bash
# submit_pipeline.sh -- submits the full download -> fit -> score-test chain

DOWNLOAD=$(qsub download_precision.pbs)
echo "Download job: $DOWNLOAD"

FIT=$(qsub -W depend=afterok:$DOWNLOAD run_graphreml_ecac.pbs)
echo "Fit job (waits for download): $FIT"

SCORE=$(qsub -W depend=afterok:$FIT run_score_test.pbs)
echo "Score test job (waits for fit): $SCORE"
```

Run `./submit_pipeline.sh` once, then walk away — all three stages queue immediately, but only execute in order, and only if each prior stage genuinely succeeds.

---

*Compiled from an active graphREML/graphld PBS troubleshooting session on Avalon (QIMR Berghofer). Commands reflect PBS Professional syntax — some flags may differ on Torque/OpenPBS systems.*
