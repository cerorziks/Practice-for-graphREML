# HPC Command-Line Cheat Sheet

Job monitoring, file transfer, text processing, and other essentials for working on a PBS Professional cluster. Commands verified against real usage during an active HPC session.

> In this file, anything written as `<<<PLACEHOLDER>>>` is meant to be replaced with your own value (e.g. paths, filenames, job IDs, category names).

---

## 1. Submitting and Checking Jobs

### `qsub` — submit a job

```bash
qsub <<<myscript.pbs>>>
qsub -l ncpus=<<<1>>>,mem=<<<22gb>>>,walltime=<<<2:00:00>>> <<<myscript.pbs>>>
qsub -q <<<bigmem>>> <<<myscript.pbs>>>
qsub -I -l mem=<<<1gb>>>,walltime=<<<8:00:00>>>
```

**What to customize:**

- `<<<myscript.pbs>>>` → your PBS script filename.
- `<<<1>>>`, `<<<22gb>>>`, `<<<2:00:00>>>` → CPUs, memory, and walltime you actually need.
- `<<<bigmem>>>` → the name of the queue you use (e.g. `normal`, `bigmem`, `gpu`, etc.).

---

### `qstat` — check job/queue status

```bash
qstat -u $USER
qstat -u $USER -1n
qstat -f <<<jobid>>>
qstat -fx <<<jobid>>>
qstat -x <<<jobid>>>
qstat -H <<<jobid>>>
qstat -Q
qstat -Qf <<<queue>>>
```

**What to customize:**

- `<<<jobid>>>` → your job ID (e.g. `12345678.hpcpbs02`).
- `<<<queue>>>` → the queue name you want details for.

---

## 2. Detailed Job Monitoring

### See real resource usage vs. what you requested

```bash
qselect -u $USER -s R | xargs qstat -f | egrep "^Job|resources_used.cpupercent|resources_used.mem|resources_used.walltime|exec_vnode"
```

No placeholders here; this is generic for all your jobs.

---

### `jobperf` — live resource dashboard

```bash
module load jobperf/0.1.0
jobperf -http -http-disable-auth -w <<<jobid>>>
```

**What to customize:**

- `<<<jobid>>>` → your running job ID (without any extra hostname suffix).

**For long jobs, auto‑log resource use:**

```bash
# Inside your PBS script, after module loads:
module load jobperf/0.1.0
jobperf -record -w -rate <<<30s>>> -http-disable-auth -http &
```

**What to customize:**

- `<<<30s>>>` → how often to record stats (e.g. `10s`, `60s`, `5m`).

---

## 3. Controlling Jobs

| Command | What it does | Example |
|--------|--------------|---------|
| `qalter` | Change settings of a queued/running job | `qalter -l walltime=<<<72:00:00>>> <<<jobid>>>` |
| `qdel` | Cancel/kill a job | `qdel <<<jobid>>>` |
| `qhold` | Pause a queued job | `qhold <<<jobid>>>` |
| `qrls` | Release a held job | `qrls <<<jobid>>>` |
| `qsig -s suspend` | Pause a running job | `qsig -s suspend <<<jobid>>>` |
| `qsig -s resume` | Resume a paused job | `qsig -s resume <<<jobid>>>` |

**What to customize:**

- `<<<jobid>>>` → your job ID.
- `<<<72:00:00>>>` → new walltime you want.

### Kill or filter jobs by criteria

```bash
qselect -u $USER | xargs qdel
qselect -ts.gt.<<<01210900>>> -ts.lt.<<<01211000>>> -u $USER | xargs qdel
```

**What to customize:**

- `<<<01210900>>>`, `<<<01211000>>>` → start/end timestamps (format from your cluster’s docs).

---

## 4. File Transfer (`scp`, `rsync`)

### `scp` — copy files securely

```bash
scp <<<myfile.txt>>> user@hpcapp:<<< /path/to/workdir/ >>>

scp user@hpcapp:<<< /path/to/file.tsv >>> .

scp -r user@hpcapp:<<< /path/to/run_directory/ >>> .
```

**What to customize:**

- `<<<myfile.txt>>>` → your local filename.
- `<<< /path/to/workdir/ >>>`, `<<< /path/to/file.tsv >>>`, `<<< /path/to/run_directory/ >>>` → your actual paths on the cluster.

---

### `rsync` — smarter, resumable copying

```bash
rsync -avz user@hpcapp:<<< /path/to/runs/ >>> ./<<<local_runs>>>/
rsync -avzP user@hpcapp:<<< /path/to/file.tsv >>> .
```

**What to customize:**

- `<<< /path/to/runs/ >>>`, `<<< /path/to/file.tsv >>>` → cluster paths.
- `<<<local_runs>>>` → your local directory name.

---

## 5. `grep` / `egrep` — Searching Text and Logs

```bash
grep "ERROR" <<<job.err>>>
grep -i "error" <<<job.err>>>
grep -v "INFO" <<<job.out>>>
grep -c "WARNING" <<<job.out>>>
grep -n "iteration" <<<job.out>>>
```

**What to customize:**

- `<<<job.err>>>`, `<<<job.out>>>` → your actual log filenames.

### Multiple patterns (OR)

```bash
egrep "^Job|resources_used.cpupercent|resources_used.mem" <<<file.txt>>>
```

**What to customize:**

- `<<<file.txt>>>` → any text/log file you’re inspecting.

### Context lines

```bash
grep -A5 "Traceback" <<<job.err>>>
grep -B3 "ValueError" <<<job.err>>>
grep -B2 -A5 "ERROR" <<<job.err>>>
grep -m1 "Error" <<<job.err>>>
```

### Search many files / filenames

```bash
grep -rl "population" <<<data/>>>
find . -iname "<<<*pattern*>>>"
```

**What to customize:**

- `<<<data/>>>` → directory to search.
- `<<<*pattern*>>>` → filename pattern you care about.

---

## 6. `sed` — Editing Text Streams

### Basic substitution

```bash
sed 's/<<<OLD>>>/<<<NEW>>>/' <<<file.txt>>>
sed 's/<<<OLD>>>/<<<NEW>>>/g' <<<file.txt>>>
sed -i 's/<<<OLD>>>/<<<NEW>>>/g' <<<file.txt>>>
sed -i.bak 's/<<<OLD>>>/<<<NEW>>>/g' <<<file.txt>>>
```

**What to customize:**

- `<<<OLD>>>`, `<<<NEW>>>` → text to replace and its replacement.
- `<<<file.txt>>>` → your target file.

### Example: rename files using a pattern

```bash
newname=$(echo "$file" | sed -E 's/<<<old\.prefix\.(chr[^.]+)\.param1=[0-9.]+\.param2=[0-9.]+\.suffix>>>/\1.<<<new>>>.edgelist/')
```

**What to customize:**

- The big pattern `<<<old\.prefix...>>>` → adapt to your actual filename structure.
- `<<<new>>>` → whatever tag you want in the new names.

### Other common uses

```bash
sed -n '<<<5>>>,<<<10>>>p' <<<file.txt>>>
sed '/^#/d' <<<file.txt>>>
sed 's/^/<<<PREFIX_>>>/' <<<file.txt>>>
sed 's/$/<<<_SUFFIX>>>/' <<<file.txt>>>
sed 's/[[:space:]]\+/ /g' <<<file.txt>>>
```

**What to customize:**

- Line numbers, prefixes, suffixes, and filenames as needed.

---

## 7. `awk` — Column-Based Text Processing

```bash
awk '{print $<<<1>>>}' <<<file.txt>>>
awk -F',' '{print $<<<6>>>}' <<<metadata.csv>>>
awk -F',' '$<<<6>>>=="<<<EUR>>>" {print $<<<4>>>}' <<<metadata.csv>>>
awk '{sum+=$<<<3>>>} END {print sum}' <<<file.txt>>>
awk 'NR==<<<1>>>' <<<file.txt>>>
awk 'NR>1' <<<file.txt>>>
```

**What to customize:**

- Column numbers (`$1`, `$6`, etc.).
- Filenames (`<<<file.txt>>>`, `<<<metadata.csv>>>`).
- Values like `<<<EUR>>>` to match your own categories.

**Example: count variants per chromosome**

```bash
awk -F'\t' '{print $<<<1>>>}' <<<annotation_file.annot>>> | sort | uniq -c
```

---

## 8. Environment Modules

```bash
module avail
module avail <<<gpu>>>
module load <<<somepackage>>>/<<<1.2.3>>>
module list
module unload <<<somepackage>>>
module purge
```

**What to customize:**

- `<<<gpu>>>` → any module namespace you care about.
- `<<<somepackage>>>`, `<<<1.2.3>>>` → software name and version you actually use.

---

## 9. Disk Usage and Quotas

```bash
du -sh <<<data/some_directory/>>>
du -sh */ 2>/dev/null | sort -rh
df -h <<< /some/path >>>
quota -s
```

**What to customize:**

- `<<<data/some_directory/>>>` → directory you want to check.
- `<<< /some/path >>>` → any path on the filesystem you care about.

---

## 10. Persistent Sessions (`tmux`)

```bash
tmux new -s <<<sessionname>>>
tmux attach -t <<<sessionname>>>
tmux ls
```

**What to customize:**

- `<<<sessionname>>>` → a name you’ll remember (e.g. `monitor`, `reml`, `pipeline`).

---

## 11. Process Management

```bash
ps aux | grep <<<someprocess>>>
ps -u $USER
top
htop
kill <<<pid>>>
kill -9 <<<pid>>>
```

**What to customize:**

- `<<<someprocess>>>` → name of the process you’re looking for.
- `<<<pid>>>` → process ID (from `ps`/`top`).

---

## 12. Archiving and Compression

```bash
tar -xzf <<<archive.tar.gz>>> -C <<<target_dir/>>>
tar -czf <<<archive.tar.gz>>> <<<mydir/>>>
curl -L -o - <<<URL>>> | tar -xz -C <<<target_dir/>>>
gunzip <<<file.gz>>>
zcat <<<file.gz>>> | head
```

**What to customize:**

- `<<<archive.tar.gz>>>`, `<<<target_dir/>>>`, `<<<mydir/>>>`, `<<<URL>>>`, `<<<file.gz>>>` → your actual filenames and URLs.

---

## 13. PBS Environment Variables and Troubleshooting

### Useful variables inside a job

These are automatic; you don’t customize them, but you can print them:

```bash
echo $PBS_JOBID
echo $PBS_O_WORKDIR
echo $PBS_ARRAY_INDEX
echo $PBS_NODEFILE
```

### Passing your own variables into a job

```bash
export <<<FOO>>>=<<<bar>>>
qsub -v <<<FOO>>> <<<myscript.pbs>>>
qsub -V <<<myscript.pbs>>>
```

**What to customize:**

- `<<<FOO>>>`, `<<<bar>>>` → your variable name and value.
- `<<<myscript.pbs>>>` → your script.

### Debugging a failed job

```bash
tracejob <<<jobid>>>
cat /var/spool/PBS/spool/<<<jobid>>>.OU
cat /var/spool/PBS/spool/<<<jobid>>>.ER
```

**What to customize:**

- `<<<jobid>>>` → the job you’re debugging.

---

## 14. Practical Combined Recipes

### Watch a job’s log and resource use

```bash
tail -f <<<runs/run1/job.out>>> &
watch -n 60 'qselect -u $USER -s R | xargs qstat -f | egrep "^Job|resources_used"'
```

**What to customize:**

- `<<<runs/run1/job.out>>>` → path to your job’s output log.

### Find the first error in a log

```bash
grep -B2 -A10 -m1 "Error\|Traceback" <<<runs/run1/job.err>>>
grep -c "Error" <<<runs/run1/job.err>>>
```

### Batch-rename files

```bash
for file in *; do
  newname=$(echo "$file" | sed -E 's/<<<OLD_PATTERN>>>/<<<NEW_PATTERN>>>/')
  if [ "$newname" != "$file" ]; then
    mv "$file" "$newname"
  fi
done
```

**What to customize:**

- `<<<OLD_PATTERN>>>`, `<<<NEW_PATTERN>>>` → your actual find/replace patterns.

### Count files per category

```bash
for pop in <<<CAT1>>> <<<CAT2>>> <<<CAT3>>> <<<CAT4>>>; do
  n=$(ls <<<data/dir/>>>/*."$pop".edgelist 2>/dev/null | wc -l)
  echo "$pop: $n files"
done
```

**What to customize:**

- `<<<CAT1>>>`… → your real category names (e.g. `AFR`, `EUR`, etc., or anything else).
- `<<<data/dir/>>>` → your data directory.
- `*.edgelist` → change extension if your files use something else.

### Get unique values from one CSV column

```bash
cut -d',' -f<<<6>>> <<<data/metadata.csv>>> | sort -u
```

**What to customize:**

- `<<<6>>>` → column number.
- `<<<data/metadata.csv>>>` → your CSV file.

---

## 15. Git Quick Reference

```bash
git status
git diff
git add <<<file.pbs>>>
git add -A
git commit -m "<<<message>>>"
git log --oneline -<<<10>>>
git describe --tags
git pull
git push
git checkout -b <<<my-branch>>>
```

**What to customize:**

- `<<<file.pbs>>>` → whatever file(s) you’re tracking.
- `<<<message>>>` → your commit message.
- `<<<10>>>` → how many recent commits to show.
- `<<<my-branch>>>` → your new branch name.

### Find exact code version

```bash
git log -1 --format="%H %ci"
git describe --tags
```

No placeholders; these just report info.

---

## 16. Bash Productivity Tips

### History and repetition

```bash
!!
!$
!<<<<n>>>
Ctrl+R
history | grep "qsub"
```

**What to customize:**

- `<<<n>>>` → history entry number (from `history`).

### Navigation shortcuts

No placeholders; these are general shortcuts.

### Background jobs in your shell

```bash
<<<long_command>>> &
Ctrl+Z
bg
fg
jobs
```

**What to customize:**

- `<<<long_command>>>` → whatever command you want to run in the background.

### Aliases (shortcuts)

Add to `~/.bashrc`, then run `source ~/.bashrc`:

```bash
alias ll='ls -lh'
alias myq='qstat -u $USER -1n'
alias gs='git status'
```

You can add your own:

```bash
alias <<<myalias>>>=<<<'some-long-command'>>>
```

### Brace expansion

```bash
touch <<<README.txt>>> <<<requirements.txt>>> <<<TODO.txt>>>
# vs.
touch {<<<README>>>,<<<requirements>>>,<<<TODO>>}.txt

mkdir <<<project>>> && cd <<<project>>>
# vs.
mkdir <<<project>>> && cd $_
```

**What to customize:**

- Filenames and directory names as you like.

---

## 17. PBS Job Chaining and Dependencies

### Basic chain

```bash
JOB1=$(qsub <<<01_step1.pbs>>>)
JOB2=$(qsub -W depend=afterok:$JOB1 <<<02_step2.pbs>>>)
JOB3=$(qsub -W depend=afterok:$JOB2 <<<03_step3.pbs>>>)
```

**What to customize:**

- `<<<01_step1.pbs>>>`, `<<<02_step2.pbs>>>`, `<<<03_step3.pbs>>>` → your actual script names.

### Example: generic pipeline

```bash
#!/bin/bash
# submit_pipeline.sh

STEP1=$(qsub <<<01_step1.pbs>>>)
echo "Step 1 job: $STEP1"

STEP2=$(qsub -W depend=afterok:$STEP1 <<<02_step2.pbs>>>)
echo "Step 2 job (waits for step 1): $STEP2"

STEP3=$(qsub -W depend=afterok:$STEP2 <<<03_step3.pbs>>>)
echo "Step 3 job (waits for step 2): $STEP3"
```

---

Compiled from an active PBS troubleshooting session on a PBS Professional cluster. Commands reflect PBS Professional syntax — some flags may differ on Torque/OpenPBS systems.
