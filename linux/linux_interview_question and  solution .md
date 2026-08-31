+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
1)Managing High I/O Processes
Company: Revolut | Difficulty: Easy
Scenario
Users are complaining about slow file access. System metrics show high disk utilization.
Task
Reduce `I/O` activity of top offender using I/O priorities to `idle`. Keep critical jobs (databases, message queues, applications) at high priority.
Example

```
 # Before (high I/O contention)

 Total DISK READ:      45.67 M/s | Total DISK WRITE:     123.45 M/s

   TID  PRIO  USER      DISK READ  DISK WRITE  COMMAND
  5678  be/4  backup     2.34 M/s   98.76 M/s  rsync /data /backup
  3421  be/4  postgres  15.23 M/s   12.45 M/s  postgres: vacuum
  8234  be/4  appuser    8.45 M/s    3.21 M/s  /usr/bin/log-processor

 # After (non-critical jobs throttled)

 Total DISK READ:      18.34 M/s | Total DISK WRITE:      35.21 M/s

   TID  PRIO  USER      DISK READ  DISK WRITE  COMMAND
  5678  idle  backup     0.50 M/s    2.10 M/s  rsync /data /backup
  3421  be/4  postgres  15.23 M/s   12.45 M/s  postgres: vacuum
  8234  idle  appuser    1.20 M/s    0.80 M/s  /usr/bin/log-processor

```
Ans 
# How to Answer "Managing High I/O Processes" in an Interview

---

## The Golden Framework — STAR + Technical Depth

Interviewers at companies like Revolut want **two things simultaneously:**
- You can **think through a problem** (not just memorize commands)
- You know the **actual tools** and can explain them clearly

```
S — Situation   (what was happening)
T — Task        (what you needed to do)
A — Action      (exactly what you did, step by step)
R — Result      (measurable outcome)
```

---

## Model Answer — Say Exactly This

> **Interviewer:** *"Users are complaining about slow file access. How do you handle it?"*

---

### Opening — Show You Think Before You Act
> *"Before touching anything, I'd first confirm whether it's actually a disk I/O problem or something else — could be CPU, memory pressure, or even a network-mounted filesystem."*

```bash
# What you'd mention running first:
iostat -x 1 5      # check disk utilization %
vmstat 1 5         # rule out memory/CPU
df -h              # rule out full disk
```

> *"If `iostat` shows `%util` near 100%, I know the disk is saturated — that confirms I/O contention."*

---

### Identification — Show You Find Root Cause
> *"Next I'd use `iotop` to find which process is the top offender."*

```bash
sudo iotop -bon 2 --only
```

> *"I'm looking at the DISK WRITE column specifically — in this case rsync is writing 98 MB/s which is clearly the culprit. It's a backup job competing equally with postgres for disk bandwidth."*

---

### Core Answer — The Technical Meat

> *"Linux uses an I/O scheduler — by default every process runs at `best-effort` class, meaning they all compete fairly for disk. The fix is `ionice`."*

> *"I'd set the rsync process to `idle` class:"*

```bash
sudo ionice -c 3 -p 5678
```

> *"Idle class means the process only gets disk access when nothing else needs it. It doesn't kill rsync — it just makes it politely step aside for critical processes. rsync still completes, just without starving the database."*

> *"Then I'd explicitly protect postgres and any message queues at best-effort highest priority:"*

```bash
sudo ionice -c 2 -n 0 -p $(pgrep -x postgres)
```

---

### Production Thinking — This Impresses Senior Interviewers

> *"But `ionice` on a PID is temporary — it resets if the service restarts. In production I'd make it permanent via systemd:"*

```ini
# For backup service
IOSchedulingClass=idle

# For postgres
IOSchedulingClass=best-effort
IOSchedulingPriority=0
```

> *"I'd also combine it with `nice -n 19` for the backup job to throttle CPU as well, since backups can be greedy on both resources."*

---

### Result — Close Strong
> *"After this, disk write dropped from 123 MB/s to ~35 MB/s total, postgres was unaffected, and user-facing file access returned to normal — all without stopping the backup job."*

---

## What Interviewers Are Actually Evaluating

| What They Ask | What They're Really Testing |
|---|---|
| "How do you fix high I/O?" | Do you diagnose first or jump to solutions? |
| "What is ionice?" | Do you understand schedulers, not just commands? |
| "How do you protect databases?" | Do you think about blast radius? |
| "What about after restart?" | Do you think about production durability? |

---

## Common Follow-up Questions + Answers

**Q: "What's the difference between `nice` and `ionice`?"**
> *"`nice` controls CPU scheduling priority. `ionice` controls disk I/O scheduling priority. They're independent — a process can be CPU-niced but still hammer the disk, so for backup jobs I use both together."*

---

**Q: "Does `ionice` always work?"**
> *"Good question — `ionice` works best with the `cfq` or `bfq` I/O schedulers. On modern systems with NVMe drives using the `none` scheduler, ionice has limited effect because the hardware itself handles queuing. In that case, I'd look at cgroups v2 `io.weight` for proper throttling."*

This answer alone puts you in the **top 10%** of candidates.

---

**Q: "What if you can't restart the backup job?"**
> *"No problem — `ionice -p <PID>` changes the priority of a running process live, no restart needed. That's actually the first thing I'd do — immediate relief without any service disruption."*

---

**Q: "Any other tools for I/O management?"**
> *"Yes — for more granular control, cgroups v2 lets you set hard I/O limits in bytes/sec per service, not just priorities. Something like `io.max` in a cgroup can cap a backup job at say 20 MB/s regardless of available bandwidth. That's more infrastructure-heavy but gives you hard guarantees."*

---

## What NOT to Say ❌

| Don't Say | Why It's Bad |
|---|---|
| *"I'd kill the rsync process"* | Destroys the backup — you didn't think about impact |
| *"I'd restart the server"* | Nuclear option for a simple problem |
| *"I'd just run ionice"* | No diagnosis step — looks like you memorized commands |
| *"I'm not sure, I'd Google it"* | Fine in real life, not in an interview |
| Jumping straight to commands | Shows no structured thinking |

---

## The One-Liner Summary to Open or Close With

> *"My approach is always: **measure first, identify the offender, throttle non-critical processes with ionice idle class, protect critical ones at best-effort high priority, and make it permanent via systemd** — so the fix survives the next deployment."*

---

**This answer works because:** you showed diagnosis → root cause → fix → blast radius awareness → production durability. That's senior-level thinking even for an "Easy" difficulty question. At Revolut specifically, they care a lot about **not breaking production** — so the "protect critical jobs first" framing scores extra points.
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
2) Analyzing Log Partition Usage
Company: RedHat | Difficulty: Medium
Scenario
Log rotation has stopped working correctly, and you suspect that `/var/log` might be mounted on a different filesystem with limited space or incorrect mount options.
Task
Determine which filesystem or device `/var/log` is mounted on, including device name, mount point, filesystem type, size, and usage. Save the findings to `/home/devops/varlog_filesystem_info.txt` Optionally identify if this filesystem differs from `/` which could provide additional info on causes of log rotation or space issues.
Example

```
# After (filesystem identified and analyzed)

Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       10G   9.8G  200M  98% /var/log

Mount details:
/dev/sdb1 on /var/log type ext4 (rw,relatime)

```

## Analyzing Log Partition Usage — Full Breakdown

---

## Understanding the Problem First

```
/                          ← root filesystem (could be /dev/sda1)
├── /home
├── /tmp
└── /var
    └── /var/log           ← might be a SEPARATE mount (/dev/sdb1)
                              with its own size, type, options
```

When `/var/log` is on a **separate partition**, it can have:
- Its own size limit (fills up independently of `/`)
- Different mount options (`noexec`, `ro`, `nosuid`)
- Different filesystem type (`xfs` vs `ext4`) — affects log rotation behavior

---

## The Commands — Each One Explained

### 1. `df` — Disk Free (space usage)

```bash
df -hT /var/log
```

```
Filesystem    Type   Size  Used  Avail  Use%  Mounted on
/dev/sdb1     ext4   10G   9.8G  200M   98%   /var/log
```

| Flag | Meaning |
|------|---------|
| `-h` | Human-readable (GB/MB instead of bytes) |
| `-T` | Show filesystem **Type** (ext4, xfs, tmpfs) |
| `/var/log` | Target path — df auto-resolves to its device |

> `df` answers: **how full is it?**

---

### 2. `findmnt` — Find Mount (best tool for this task)

```bash
findmnt /var/log
```

```
TARGET    SOURCE    FSTYPE  OPTIONS
/var/log  /dev/sdb1 ext4    rw,relatime,data=ordered
```

```bash
# More detailed output
findmnt -o TARGET,SOURCE,FSTYPE,SIZE,USE%,OPTIONS /var/log
```

```
TARGET    SOURCE     FSTYPE  SIZE   USE%  OPTIONS
/var/log  /dev/sdb1  ext4    10G    98%   rw,relatime
```

> `findmnt` answers: **what is mounted where, how, and with what options?**

---

### 3. `lsblk` — List Block Devices (device-level view)

```bash
lsblk -f /dev/sdb1
```

```
NAME  FSTYPE  LABEL  UUID                                  MOUNTPOINT
sdb1  ext4           a1b2c3d4-e5f6-7890-abcd-ef1234567890  /var/log
```

> `lsblk` answers: **what's the block device behind this mount?**

---

### 4. `mount` — Raw Mount Table

```bash
mount | grep "/var/log"
```

```
/dev/sdb1 on /var/log type ext4 (rw,relatime)
```

Or read directly from kernel:
```bash
cat /proc/mounts | grep "var/log"
```

---

### 5. Compare `/var/log` vs `/` — Key Part of the Task

```bash
df -hT / /var/log
```

```
Filesystem  Type   Size   Used  Avail  Use%  Mounted on
/dev/sda1   xfs    50G    20G   30G    40%   /
/dev/sdb1   ext4   10G    9.8G  200M   98%   /var/log
```

If both show the **same device** → `/var/log` is NOT separately mounted (shares `/`).
If **different devices** → separately mounted partition confirmed.

---

## Full Script — Save Findings to File

```bash
#!/bin/bash
# Script: varlog_analyze.sh
# Purpose: Analyze /var/log filesystem and save report

OUTPUT="/home/devops/varlog_filesystem_info.txt"

{
    echo "========================================"
    echo "  /var/log Filesystem Analysis Report"
    echo "  Generated: $(date)"
    echo "========================================"
    echo ""

    echo "--- DISK USAGE ---"
    df -hT /var/log
    echo ""

    echo "--- MOUNT DETAILS ---"
    findmnt -o TARGET,SOURCE,FSTYPE,SIZE,USE%,OPTIONS /var/log
    echo ""

    echo "--- RAW MOUNT ENTRY ---"
    mount | grep "var/log"
    echo ""

    echo "--- BLOCK DEVICE INFO ---"
    DEVICE=$(df /var/log | awk 'NR==2 {print $1}')
    lsblk -f "$DEVICE"
    echo ""

    echo "--- COMPARISON: / vs /var/log ---"
    df -hT / /var/log
    echo ""

    # Check if same or different filesystem
    ROOT_DEV=$(df / | awk 'NR==2 {print $1}')
    LOG_DEV=$(df /var/log | awk 'NR==2 {print $1}')

    echo "--- VERDICT ---"
    if [ "$ROOT_DEV" == "$LOG_DEV" ]; then
        echo "SAME filesystem: /var/log shares device $ROOT_DEV with /"
        echo "Log rotation issues are likely due to overall disk space."
    else
        echo "DIFFERENT filesystem detected!"
        echo "  / is on      : $ROOT_DEV"
        echo "/var/log is on : $LOG_DEV"
        echo "Check: mount options, size limits, and filesystem type."
    fi

    echo ""
    echo "--- MOUNT OPTIONS CHECK ---"
    OPTS=$(findmnt -no OPTIONS /var/log)
    echo "Options: $OPTS"

    # Flag problematic options
    echo "$OPTS" | grep -q "ro"      && echo "⚠ WARNING: Mounted READ-ONLY — logrotate will fail!"
    echo "$OPTS" | grep -q "noexec"  && echo "⚠ WARNING: noexec set — scripts in /var/log cannot execute"
    echo "$OPTS" | grep -q "nosuid"  && echo "INFO: nosuid set (normal for log partitions)"

} | tee "$OUTPUT"

echo ""
echo "Report saved to: $OUTPUT"
```

Run it:
```bash
chmod +x varlog_analyze.sh
sudo ./varlog_analyze.sh
```

---

## Sample Output File

```
========================================
  /var/log Filesystem Analysis Report
  Generated: Sat May 02 10:32:11 IST 2026
========================================

--- DISK USAGE ---
Filesystem  Type  Size  Used  Avail  Use%  Mounted on
/dev/sdb1   ext4  10G   9.8G  200M   98%   /var/log

--- MOUNT DETAILS ---
TARGET    SOURCE     FSTYPE  SIZE   USE%  OPTIONS
/var/log  /dev/sdb1  ext4    10G    98%   rw,relatime

--- RAW MOUNT ENTRY ---
/dev/sdb1 on /var/log type ext4 (rw,relatime)

--- BLOCK DEVICE INFO ---
NAME  FSTYPE  UUID                                  MOUNTPOINT
sdb1  ext4    a1b2c3d4-e5f6-7890-abcd-ef1234567890  /var/log

--- COMPARISON: / vs /var/log ---
Filesystem  Type  Size  Used  Avail  Use%  Mounted on
/dev/sda1   xfs   50G   20G   30G    40%   /
/dev/sdb1   ext4  10G   9.8G  200M   98%   /var/log

--- VERDICT ---
DIFFERENT filesystem detected!
  / is on      : /dev/sda1
/var/log is on : /dev/sdb1
Check: mount options, size limits, and filesystem type.

--- MOUNT OPTIONS CHECK ---
Options: rw,relatime
```

---

## Why Log Rotation Breaks — Root Cause Mapping

| Finding | Root Cause | Fix |
|---|---|---|
| `Use% = 98%` | Partition full — no space for new logs | `logrotate` compress + extend partition |
| Mounted `ro` (read-only) | logrotate can't write/delete | Fix `/etc/fstab`, remount `rw` |
| `noexec` set | Post-rotate scripts fail silently | Remove `noexec` if scripts needed |
| `ext4` but `xfs` on `/` | Different filesystem behavior | Tune logrotate config per fs type |
| Same device as `/` | `/` filling up affects `/var/log` | Add dedicated partition |

---

## Quick One-Liners (for interview or real use)

```bash
# Single command to get everything important
findmnt -o TARGET,SOURCE,FSTYPE,SIZE,USE%,OPTIONS /var/log

# Check if /var/log is on its own partition
[ "$(df / | awk 'NR==2{print $1}')" != "$(df /var/log | awk 'NR==2{print $1}')" ] \
  && echo "Separate partition" || echo "Shares root"

# Save df + findmnt + mount info in one shot
{ df -hT /var/log; findmnt /var/log; mount | grep var/log; } \
  > /home/devops/varlog_filesystem_info.txt
```

---

## TL;DR — Command Priority

```
findmnt /var/log     ← use this FIRST — gives everything in one shot
df -hT /var/log      ← space usage + type
mount | grep varlog  ← raw mount options
lsblk -f             ← device-level UUID and type
```

**Core insight for RedHat interview:** The difference between `/var/log` on its own partition vs sharing `/` is critical — separate partitions protect root from filling up, but they introduce their own size constraints and mount option issues that directly cause logrotate failures.

# Analyzing Log Partition Usage — Simple + Interview Ready

---

## Part 1: Understand It Simply (Like a Story)

### What is a Filesystem?

Think of your Linux server like an **office building:**

```
OFFICE BUILDING (Your Server)
│
├── Floor 1 (/)          ← Main floor, root filesystem
├── Floor 2 (/home)      ← Where users live
└── Floor 3 (/var/log)   ← Where all logs are stored
```

Each floor can be:
- A **separate room with its own storage cabinet** (separate partition)
- Or just **part of the main floor** (same partition as `/`)

---

### Why Does This Matter for Log Rotation?

**Log rotation** = Linux automatically deletes/compresses old log files to free space.

When it **stops working**, common reasons are:

```
Reason 1: The /var/log partition is FULL (98% used)
          → No space to write new/compressed logs

Reason 2: /var/log is mounted READ-ONLY
          → logrotate tries to write → fails silently

Reason 3: /var/log is on a tiny separate partition
          → fills up fast, doesn't affect / at all
```

---

### The 3 Commands You Actually Need

```
findmnt  →  WHERE is it mounted + HOW
df       →  HOW FULL is it
mount    →  WHAT OPTIONS does it have
```

That's it. Everything else is just extra detail.

---

## Part 2: The Task — Broken Down Simply

### Step 1 — Find which device `/var/log` is on

```bash
findmnt /var/log
```
```
TARGET    SOURCE     FSTYPE  OPTIONS
/var/log  /dev/sdb1  ext4    rw,relatime
```
**Plain English:** `/var/log` lives on device `/dev/sdb1`, formatted as `ext4`, mounted read-write.

---

### Step 2 — Check how full it is

```bash
df -hT /var/log
```
```
Filesystem  Type  Size  Used  Avail  Use%  Mounted on
/dev/sdb1   ext4  10G   9.8G  200M   98%   /var/log
```
**Plain English:** 10GB total, 9.8GB used, only 200MB free. Almost full!

---

### Step 3 — Is it separate from `/`?

```bash
df -hT / /var/log
```
```
Filesystem  Type  Size  Used  Avail  Use%  Mounted on
/dev/sda1   xfs   50G   20G   30G    40%   /          ← different device!
/dev/sdb1   ext4  10G   9.8G  200M   98%   /var/log   ← separate partition
```
**Plain English:** `/` is on `sda1` (healthy, 40% used). `/var/log` is on `sdb1` (separate, almost full). Problem found!

---

### Step 4 — Save to file (the task requirement)

```bash
{
  echo "=== /var/log Analysis ==="
  echo "Date: $(date)"
  echo ""
  echo "-- Space Usage --"
  df -hT /var/log
  echo ""
  echo "-- Mount Details --"
  findmnt /var/log
  echo ""
  echo "-- Comparison with / --"
  df -hT / /var/log
} > /home/devops/varlog_filesystem_info.txt
```

---

## Part 3: Interview Questions — Detailed Answers

---

### Q1. "What is a mount point and why does `/var/log` sometimes have its own?"

**Simple Answer:**
> *"A mount point is a directory where a separate storage device or partition is attached. `/var/log` gets its own partition so that if logs fill up completely, they don't crash the entire system by filling `/`. It isolates the problem."*

**Story to tell:**
> *"Imagine `/` is your main hard drive. If logs grow out of control and share `/`, your entire OS runs out of space — SSH stops working, processes crash. Putting `/var/log` on its own partition is like giving logs their own separate drive. Worst case, logs stop — not the whole system."*

---

### Q2. "How do you find which device `/var/log` is mounted on?"

**Answer:**
> *"I use `findmnt` — it's the cleanest tool for this. One command gives me the device, filesystem type, size, and mount options all at once."*

```bash
findmnt /var/log
# OR more detailed:
findmnt -o TARGET,SOURCE,FSTYPE,SIZE,USE%,OPTIONS /var/log
```

> *"I can also use `df -hT /var/log` which shows disk usage alongside the device name, or `mount | grep var/log` to read raw mount options from the kernel."*

---

### Q3. "Log rotation stopped working. Walk me through how you'd diagnose it."

**This is the main interview question. Answer in steps:**

> **Step 1 — Check if disk is full:**
```bash
df -hT /var/log
# If Use% is 95%+ → that's your problem
```

> **Step 2 — Check mount options:**
```bash
findmnt -o OPTIONS /var/log
# Look for "ro" → read-only = logrotate can't write
# Look for "noexec" → post-rotate scripts will fail
```

> **Step 3 — Check logrotate config itself:**
```bash
cat /etc/logrotate.conf
ls /etc/logrotate.d/
# Look for wrong paths, wrong permissions
```

> **Step 4 — Run logrotate manually with debug:**
```bash
sudo logrotate -d /etc/logrotate.conf
# -d = dry run, shows exactly what it would do and any errors
```

> **Step 5 — Check if /var/log is separate from /:**
```bash
df / /var/log
# Different devices = separate partition with its own limits
```

---

### Q4. "What is `findmnt` and why is it better than just using `mount`?"

**Answer:**
> *"`mount` gives you raw output from `/proc/mounts` — it works but the output is messy and hard to filter. `findmnt` is purpose-built for querying mount points. You can target a specific path, choose exactly which columns you want, and get clean output that's easy to script with."*

```bash
# mount output — noisy
mount | grep var/log
# /dev/sdb1 on /var/log type ext4 (rw,relatime,data=ordered,errors=remount-ro)

# findmnt output — clean, targeted
findmnt /var/log
# TARGET    SOURCE     FSTYPE  OPTIONS
# /var/log  /dev/sdb1  ext4    rw,relatime
```

---

### Q5. "What mount options could cause log rotation to fail?"

**Answer — know these 3:**

| Option | What it means | How it breaks logrotate |
|--------|--------------|------------------------|
| `ro` | Read-only | logrotate can't create/delete files |
| `noexec` | Can't run executables | Post-rotate scripts fail |
| `nosuid` | No setuid bits | Permissions issues on some scripts |

> *"The most dangerous one is `ro` — read-only. Logrotate tries to rename, compress, and delete log files. If the filesystem is read-only, every single one of those operations fails silently unless you check the logrotate error log at `/var/lib/logrotate/status`."*

---

### Q6. "What's the difference between `df` and `du`? Which do you use here?"

**Answer:**
> *"`df` reports filesystem-level disk usage — it asks the filesystem itself how much space is used and free. `du` walks through directories and adds up file sizes. For this task I use `df` because I want partition-level info — device name, total size, mount point. `du` would be my next step if I wanted to find which specific log files are consuming the most space."*

```bash
df -hT /var/log          # partition level → how full is the device
du -sh /var/log/*        # directory level → which log is biggest
du -sh /var/log/* | sort -rh | head -10   # top 10 biggest logs
```

---

### Q7. "How do you check if `/var/log` is on the same filesystem as `/`?"

**Answer:**
> *"I compare the device names from `df`:"*

```bash
df / /var/log
```

> *"If both rows show the same device like `/dev/sda1`, they share the same partition. If they show different devices like `sda1` and `sdb1`, `/var/log` is on its own partition. In a script I'd do it like this:"*

```bash
ROOT_DEV=$(df / | awk 'NR==2 {print $1}')
LOG_DEV=$(df /var/log | awk 'NR==2 {print $1}')

if [ "$ROOT_DEV" = "$LOG_DEV" ]; then
    echo "Same filesystem — shared partition"
else
    echo "Different filesystem — /var/log is separate"
fi
```

---

### Q8. "The `/var/log` partition is 98% full. What do you do?"

**Answer — give 3 options showing depth:**

> **Option 1 — Immediate relief (buy time):**
```bash
# Find and delete old/compressed logs
find /var/log -name "*.gz" -mtime +30 -delete
find /var/log -name "*.log" -size +100M
```

> **Option 2 — Force logrotate to run now:**
```bash
sudo logrotate -f /etc/logrotate.conf
```

> **Option 3 — Permanent fix (resize partition):**
```bash
# If using LVM (most production systems):
lvextend -L +5G /dev/vg0/var_log
resize2fs /dev/vg0/var_log     # ext4
# OR
xfs_growfs /var/log            # xfs
```

> *"I'd also investigate WHY it filled up — which service is logging excessively — and fix the logrotate config for that service so it doesn't happen again."*

---

## Part 4: The One Perfect Answer to Open With

If the interviewer asks the broad question:

> *"Log rotation has stopped. `/var/log` might be on a separate filesystem. How do you investigate?"*

**Say this:**

> *"My first move is `findmnt /var/log` — single command that tells me the device, filesystem type, and mount options. Then `df -hT /var/log` to see if it's full. Then I compare it against `/` with `df / /var/log` to confirm it's a separate partition. The three things that break logrotate on a separate partition are: it's full, it's mounted read-only, or it has `noexec` set. I check all three, save the findings to a file, then fix based on what I find."*

---

## Part 5: Cheat Sheet

```
FIND WHERE IT'S MOUNTED:
  findmnt /var/log

CHECK IF IT'S FULL:
  df -hT /var/log

CHECK MOUNT OPTIONS:
  findmnt -no OPTIONS /var/log

COMPARE WITH ROOT:
  df -hT / /var/log

SAVE TO FILE:
  { df -hT /var/log; findmnt /var/log; } > /home/devops/varlog_filesystem_info.txt

TOP 3 CAUSES OF LOGROTATE FAILURE:
  1. Partition full (Use% = 98%+)
  2. Mounted read-only (ro in options)
  3. noexec mount option set
```

---

**Key mindset for RedHat interviews:** They don't just want commands — they want you to explain **why** the problem happens and show you understand the relationship between partitions, mount options, and how tools like logrotate interact with the filesystem. The story matters as much as the syntax.

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
3) Automated Archive and Retention
Company: Microsoft | Difficulty: Hard
Scenario
Configuration files in `/etc` are at risk of being lost due to accidental changes or deletions, and there's currently no automated backup process in place.
Task
Write a shell script at `/usr/local/bin/backup_etc.sh` that accepts a target backup path (where files will be saved at) as a command-line argument, creates a compressed archive of `/etc` with the naming format `etc-backup-YYYY-MM-DD.tar.gz`, automatically removes backups older than 7 days, and exits with an error if no path is provided. Make the script executable and create a cron job to run it daily at 02:00 AM, storing backups in `/backups/etc/` using `crontab` command. You can use __https://crontab.guru__ for cronjob format.
Once script is created execute it `/usr/local/bin/backup_etc.sh /backups/etc/`
Example

```
# Before (no automated backups)

No backup script exists
/etc directory unprotected
Manual backups required

```


```
# After (automated backup system configured)

Backup script created and executable
Running without argument:
  Error: Backup directory path required

Running with argument creates timestamped backup:
  /backups/etc/etc-backup-2025-11-06.tar.gz

After 7 days of daily backups:
  etc-backup-2025-11-01.tar.gz (deleted - older than 7 days)
  etc-backup-2025-11-02.tar.gz (deleted - older than 7 days)
  etc-backup-2025-11-03.tar.gz
  etc-backup-2025-11-04.tar.gz
  etc-backup-2025-11-05.tar.gz
  etc-backup-2025-11-06.tar.gz
  etc-backup-2025-11-07.tar.gz
  etc-backup-2025-11-08.tar.gz
  etc-backup-2025-11-09.tar.gz

Cron job configured: runs daily at 02:00 AM
```
## Let's Build It First — Then Understand It

---✅ Everything working. Now let's understand it deeply.

---

## Part 1: Understand the Script — Line by Line (Simply)

### The Big Picture

```
User runs script
      ↓
Did they give a backup path?
      ↓ NO → print error, exit
      ↓ YES
      ↓
Create the folder if it doesn't exist
      ↓
Compress /etc into a .tar.gz file with today's date
      ↓
Delete any backup older than 7 days
      ↓
Show what's left
```

---

### Every Concept Explained

#### `$1` — What is a command-line argument?

```bash
/usr/local/bin/backup_etc.sh /backups/etc/
                              ↑
                        This is $1
```

`$1` = first word the user types after the script name. Like a parameter you pass to a function.

```bash
if [ -z "$1" ]; then   # -z means "is this empty/zero length?"
    echo "Error: No path given"
    exit 1             # exit with code 1 = failure
fi
```

---

#### `date +%Y-%m-%d` — Why this format?

```bash
DATE=$(date +%Y-%m-%d)
# Output: 2026-05-02
```

| Symbol | Meaning | Output |
|--------|---------|--------|
| `%Y` | 4-digit year | 2026 |
| `%m` | 2-digit month | 05 |
| `%d` | 2-digit day | 02 |

Why this format? Because files sort **alphabetically** and this format sorts **chronologically** too:
```
etc-backup-2026-04-30.tar.gz   ← oldest
etc-backup-2026-05-01.tar.gz
etc-backup-2026-05-02.tar.gz   ← newest
```

---

#### `tar -czf` — What does each flag mean?

```bash
tar -czf "$FULL_PATH" /etc
```

| Flag | Meaning |
|------|---------|
| `c` | **C**reate a new archive |
| `z` | Compress with g**z**ip (.gz) |
| `f` | Next argument is the **f**ile name |

```
tar -czf  /backups/etc/etc-backup-2026-05-02.tar.gz  /etc
  ↑              ↑                                      ↑
create+compress  save here                       from this directory
```

`2>/dev/null` — hides permission warnings (normal when backing up `/etc` as non-root)

---

#### `$?` — How do you know if a command succeeded?

```bash
tar -czf "$FULL_PATH" /etc 2>/dev/null

if [ $? -eq 0 ]; then
    echo "Backup created successfully"
else
    echo "Error: Backup failed!"
    exit 1
fi
```

`$?` = exit code of the **last command**
- `0` = success (in Linux, zero always means OK)
- Non-zero = failure

---

#### `find` — How does retention/cleanup work?

```bash
find "$BACKUP_DIR" -name "etc-backup-*.tar.gz" -mtime +7 -delete
```

| Part | Meaning |
|------|---------|
| `"$BACKUP_DIR"` | Look in this folder |
| `-name "etc-backup-*.tar.gz"` | Match only our backup files |
| `-mtime +7` | Modified more than 7 days ago |
| `-delete` | Delete them |

```
Today: May 2

etc-backup-2026-04-22.tar.gz  → 10 days old → DELETED ✓
etc-backup-2026-04-23.tar.gz  → 9 days old  → DELETED ✓
etc-backup-2026-04-26.tar.gz  → 6 days old  → KEPT ✓
etc-backup-2026-04-29.tar.gz  → 3 days old  → KEPT ✓
etc-backup-2026-05-02.tar.gz  → today       → KEPT ✓
```

---

#### `chmod +x` — Making script executable

```bash
chmod +x /usr/local/bin/backup_etc.sh
```

```
Before:  -rw-r--r--   (readable, not runnable)
After:   -rwxr-xr-x   (readable + executable by everyone)
              ↑
              x = execute permission added
```

Without this: `bash: permission denied`

---

#### Cron Job — `0 2 * * *`

```
0  2  *  *  *   /usr/local/bin/backup_etc.sh /backups/etc/
↑  ↑  ↑  ↑  ↑
│  │  │  │  └── Day of week (0-7, * = every day)
│  │  │  └───── Month (* = every month)
│  │  └──────── Day of month (* = every day)
│  └─────────── Hour (2 = 2 AM)
└────────────── Minute (0 = exactly :00)
```

**Why 2 AM?** Least user activity. Server is mostly idle. Safe to do heavy disk operations.

---

## Part 2: Interview Questions — Full Detailed Answers

---

### Q1. "What does this script do? Explain it simply."

**Answer:**
> *"It's an automated backup script for `/etc` — the directory that holds all system configuration files. It takes a backup destination as an argument, creates a compressed `.tar.gz` archive named with today's date, then cleans up any backups older than 7 days. It's scheduled to run at 2 AM daily via cron so it's fully hands-off."*

---

### Q2. "Why do we back up `/etc` specifically?"

**Answer:**
> *"`/etc` contains all system and application configuration files — SSH config, network settings, cron jobs, user accounts in `/etc/passwd`, DNS settings, everything. If a junior admin accidentally edits `/etc/sudoers` incorrectly or deletes `/etc/fstab`, the system may not boot or SSH may break entirely. Having a daily backup means you can restore the exact config from yesterday. It's the most critical directory to protect on a Linux server."*

---

### Q3. "What is `tar` and why `tar.gz` specifically?"

**Answer:**
> *"`tar` stands for Tape Archive — it bundles multiple files and directories into a single file while preserving permissions, ownership, and directory structure. The `.gz` compression is added with the `-z` flag, which runs gzip on the archive. The reason we use `tar.gz` over just `tar` is purely space — `/etc` might be small, but compressed it could be 50-70% smaller. On production systems where you're keeping 7 days of backups, compression adds up."*

```bash
# tar without compression — just bundles
tar -cf etc-backup.tar /etc       # bigger

# tar with gzip compression
tar -czf etc-backup.tar.gz /etc   # smaller, same content
```

---

### Q4. "Explain the retention logic. How does `find -mtime +7` work?"

**Answer:**
> *"`find` with `-mtime +7` looks at the file's modification timestamp. `+7` means 'more than 7 days ago' — it uses 24-hour blocks from the current time. So if today is May 2nd at 2 AM, it deletes anything last modified before April 25th at 2 AM. The `-name` filter ensures we only delete our backup files and nothing else in that directory."*

> *"One subtle thing — `-mtime` checks modification time, not creation time. On most systems these are the same for fresh backups, but if someone manually `touch`es a file it could affect it. In production I'd consider `-newer` or timestamp-based naming as an alternative."*

---

### Q5. "What does `exit 1` mean and why does it matter?"

**Answer:**
> *"Exit codes are how shell scripts communicate success or failure to the system. `exit 0` means success, any non-zero means failure. This matters critically for cron jobs — if the backup fails and exits with code 1, monitoring tools like Nagios, Zabbix, or systemd can detect the failure and alert on-call engineers. If we just echoed an error and exited with 0, cron would think everything was fine and no alert would fire."*

```bash
# Script exits 1 on failure
/usr/local/bin/backup_etc.sh
echo $?   # prints 1 → monitoring sees failure
```

---

### Q6. "What is cron and how does the cron expression `0 2 * * *` work?"

**Answer:**
> *"Cron is the Linux task scheduler daemon. It reads a table called `crontab` and runs commands on schedule. The expression has 5 fields: minute, hour, day-of-month, month, day-of-week. `0 2 * * *` means: at minute 0, hour 2, every day, every month, every day of the week — so exactly 2:00 AM daily."*

```
Common patterns worth knowing:
0 2 * * *      → daily at 2 AM
0 2 * * 0      → weekly, Sunday 2 AM
0 2 1 * *      → monthly, 1st of month at 2 AM
*/5 * * * *    → every 5 minutes
0 */6 * * *    → every 6 hours
```

---

### Q7. "What if the backup script fails silently — how do you add error handling?"

**Answer:**
> *"Two ways. First, add logging to a file so you can audit runs:"*

```bash
LOGFILE="/var/log/backup_etc.log"
echo "[$(date)] Backup started" >> "$LOGFILE"

tar -czf "$FULL_PATH" /etc 2>/dev/null
if [ $? -ne 0 ]; then
    echo "[$(date)] ERROR: tar failed" >> "$LOGFILE"
    exit 1
fi
echo "[$(date)] Backup successful: $FULL_PATH" >> "$LOGFILE"
```

> *"Second, use `set -e` at the top of the script — it makes the entire script exit immediately on any error, no silent failures:"*

```bash
#!/bin/bash
set -e          # exit on any error
set -o pipefail # catch errors in pipes too
```

> *"For cron specifically, I'd also add email notification by setting `MAILTO` at the top of the crontab:"*

```bash
MAILTO=admin@company.com
0 2 * * * /usr/local/bin/backup_etc.sh /backups/etc/
```

---

### Q8. "How do you verify the backup is not corrupted?"

**Answer:**
> *"I'd add a verification step using `tar -tzf` which lists the contents of the archive without extracting. If it returns exit code 0, the archive is readable and intact:"*

```bash
# Verify archive integrity
tar -tzf "$FULL_PATH" > /dev/null 2>&1

if [ $? -eq 0 ]; then
    echo "Backup verified OK: $FULL_PATH"
else
    echo "ERROR: Backup file is corrupted!"
    rm -f "$FULL_PATH"   # delete the bad backup
    exit 1
fi
```

> *"For critical systems I'd also store an MD5 checksum alongside each backup:"*

```bash
md5sum "$FULL_PATH" > "${FULL_PATH}.md5"
# Later, to verify:
md5sum -c "${FULL_PATH}.md5"
```

---

## Part 3: The Complete Final Script (Production Grade)

```bash
#!/bin/bash
# ============================================
# backup_etc.sh — Production Grade Version
# ============================================
set -euo pipefail

BACKUP_DIR="${1:-}"
DATE=$(date +%Y-%m-%d)
BACKUP_FILE="etc-backup-${DATE}.tar.gz"
FULL_PATH="${BACKUP_DIR}/${BACKUP_FILE}"
RETENTION_DAYS=7
LOGFILE="/var/log/backup_etc.log"
SOURCE_DIR="/etc"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOGFILE"; }

# 1. Validate argument
if [ -z "$BACKUP_DIR" ]; then
    echo "Error: Backup directory path required"
    echo "Usage: $0 <backup_directory>"
    exit 1
fi

# 2. Create directory
mkdir -p "$BACKUP_DIR"
log "Backup started → $FULL_PATH"

# 3. Create archive
tar -czf "$FULL_PATH" "$SOURCE_DIR" 2>/dev/null
log "Backup created: $(du -sh "$FULL_PATH" | cut -f1)"

# 4. Verify archive
tar -tzf "$FULL_PATH" > /dev/null 2>&1
log "Backup verified OK"

# 5. Retention cleanup
find "$BACKUP_DIR" -name "etc-backup-*.tar.gz" -mtime +${RETENTION_DAYS} -delete
log "Cleanup done. Kept last ${RETENTION_DAYS} days."

log "Backup completed successfully"
```

---

## Part 4: Cheat Sheet

```
KEY CONCEPTS:
  $1              → first argument passed to script
  -z "$1"         → true if variable is empty
  exit 1          → signal failure to calling process
  date +%Y-%m-%d  → format: 2026-05-02 (sorts correctly)
  tar -czf        → create compressed archive
  tar -tzf        → verify/list archive (no extraction)
  find -mtime +7  → files older than 7 days
  chmod +x        → make script runnable
  crontab -e      → edit cron jobs
  0 2 * * *       → daily at 2:00 AM

CRON FIELDS:
  MIN HOUR DOM MON DOW
   0   2   *   *   *  = daily 2AM
```

> **Microsoft interview tip:** They care about **reliability and observability** — always mention logging, error handling with `$?`, and verification steps. A script that silently fails is worse than no script at all in production.
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
4)CPU Resource Management Priority
Company: RedHat | Difficulty: Easy
Scenario
During peak traffic hours, a background `data-processing` job that is started by user `devops` is consuming too much CPU and slowing down user-facing services. Stopping it isn't an option because it would interrupt ongoing workflows, but you need to limit its CPU usage temporarily.
Task
Reduce the process's priority to `10` while keeping it running.
Example

```
# Before (high priority process)

USER       PID %CPU %MEM    VSZ     RSS   TTY      STAT START   TIME COMMAND
devops    5432 85.3 12.4 2048576 256840   ?        RN   14:35   8:23 /usr/bin/data-processor --batch

```


```
# After (priority adjusted)

USER       PID %CPU %MEM    VSZ     RSS   TTY      STAT START   TIME COMMAND
devops    5432 45.2 12.4 2048576 256840   ?        RN   14:35   8:45 /usr/bin/data-processor --batch

```
# CPU Resource Management Priority — Full Deep Dive

---

## Part 1: Understand It Simply (The Story)

### What is CPU Priority?

Your Linux server runs **many processes at the same time** but has limited CPU cores. The kernel has to decide:

> *"Which process gets CPU time first? How much does each get?"*

Think of it like a **queue at a bank:**

```
WITHOUT priority (everyone equal):
  data-processor  ████████████████  (eating 85% CPU)
  nginx           ████              (starved, slow)
  postgres        ████              (starved, slow)
  ssh             ██                (barely running)

WITH priority (background job deprioritized):
  nginx           ████████████      (user-facing, gets more)
  postgres        ████████          (database, healthy)
  data-processor  ████              (still runs, just slower)
  ssh             ████              (responsive again)
```

The process doesn't get **killed** — it just gets **less CPU attention** when others need it.

---

### What is "Nice Value"?

**Nice value** = how "nice" a process is to other processes.

```
Scale: -20  ←────────────────────────→  +19
        ↑                                  ↑
   GREEDY                              GENEROUS
(highest priority)                (lowest priority)
(not nice at all)               (very nice to others)

Default for all processes = 0
```

Think of it as **selfishness level:**
- `-20` = completely selfish, grabs CPU first always
- `0` = normal, fair share
- `+19` = completely selfless, takes only what's left over

**In this task:** We set nice value to `+10` — the process becomes generous, yields CPU to more important processes.

---

### Two Commands You Need

```
nice    →  set priority when STARTING a process
renice  →  change priority of ALREADY RUNNING process
```

Since the `data-processing` job is **already running** → we use `renice`.

---

## Part 2: The Commands — Fully Explained

### Check Current Priority First

```bash
# See the process and its current nice value
ps aux | grep data-processor
```
```
USER     PID  %CPU %MEM  STAT  NI   COMMAND
devops  5432  85.3 12.4  RN    0    /usr/bin/data-processor --batch
                               ↑
                         NI=0, default priority
```

```bash
# Better view — shows NI (nice) column clearly
ps -eo pid,user,ni,stat,%cpu,comm | grep data-processor
```
```
  PID USER      NI STAT %CPU COMMAND
 5432 devops     0 RN   85.3 data-processor
```

```bash
# Or use top — press 'r' to renice interactively
top -p 5432
```

---

### The Fix — `renice`

```bash
# Syntax:
renice -n <nice_value> -p <PID>

# Our command:
renice -n 10 -p 5432
```
```
5432 (process ID) old priority 0, new priority 10
```

**What just happened:**
```
Before:  NI=0   → normal priority → competes equally for CPU
After:   NI=10  → lower priority → yields CPU to other processes
```

---

### Verify It Worked

```bash
ps -eo pid,user,ni,stat,%cpu,comm | grep data-processor
```
```
  PID USER      NI STAT %CPU COMMAND
 5432 devops    10 RN   45.2 data-processor
                  ↑       ↑
              NI=10    CPU dropped from 85% to 45%
```

```bash
# Watch it live
top -p 5432
# In top output, NI column now shows 10
```

---

### What if You Need Root? (Important Nuance)

```bash
# Regular user can only increase nice (make LESS priority)
renice -n 10 -p 5432        # works as devops user ✓
renice -n -5 -p 5432        # FAILS — needs root ✗

# Root can do anything
sudo renice -n 10 -p 5432   # always works ✓
sudo renice -n -5 -p 5432   # only root can decrease nice ✓
```

**Rule:** Any user can make their process **nicer** (higher number). Only root can make a process **less nice** (lower/negative number).

---

### Setting Priority at Launch (nice)

If you're starting the job and want it pre-throttled:

```bash
# Start with nice value 10 from the beginning
nice -n 10 /usr/bin/data-processor --batch

# Start as devops user with low priority
sudo -u devops nice -n 10 /usr/bin/data-processor --batch
```

---

## Part 3: How the Kernel Actually Uses Nice Value

### The CFS (Completely Fair Scheduler)

Linux uses **CFS — Completely Fair Scheduler**. It doesn't use nice values directly — it converts them to **weight values**:

```
Nice  -20  →  weight 88761   (gets ~100x more CPU than nice 19)
Nice    0  →  weight  1024   (baseline)
Nice   10  →  weight   110   (gets ~9x LESS than default)
Nice   19  →  weight    15   (almost nothing)
```

Practical effect with 2 processes competing:

```
Process A (NI=0,  weight=1024)
Process B (NI=10, weight=110)

Total weight = 1024 + 110 = 1134

Process A gets: 1024/1134 = 90% of CPU
Process B gets:  110/1134 = 10% of CPU
```

**This is why CPU dropped from 85% to 45%** — as soon as other processes need CPU, data-processor yields.

---

### STAT Column Meanings

```bash
ps aux | grep data-processor
# STAT column shows: RN
```

| STAT code | Meaning |
|-----------|---------|
| `R` | Running — actively using CPU |
| `S` | Sleeping — waiting for something |
| `N` | Nice — has positive nice value (>0) |
| `<` | High priority — negative nice value |
| `D` | Uninterruptible sleep (I/O wait) |
| `Z` | Zombie — finished but not cleaned up |

After `renice +10`:
```
Before: RN  (Running, Normal priority)
After:  RN  (Running, Nice — N appears because NI > 0)
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "What is a nice value and what range does it have?"

**Answer:**
> *"The nice value controls a process's CPU scheduling priority in Linux. It ranges from -20 to +19. A lower (more negative) value means higher priority — the process gets more CPU time. A higher (more positive) value means lower priority — it yields to other processes. The default for all processes is 0. The name comes from the idea that a high-nice process is being 'nice' to other processes by giving up CPU time."*

```
-20  →  highest priority (most aggressive)
  0  →  default (fair share)
+19  →  lowest priority (most yielding)
```

---

### Q2. "What's the difference between `nice` and `renice`?"

**Answer:**
> *"`nice` is used when **launching** a new process — you set its priority from the start. `renice` is used to change the priority of a **process that's already running** — you target it by PID. In this scenario, the data-processing job is already running, so `renice` is the right tool. `nice -n 10 command` versus `renice -n 10 -p PID`."*

---

### Q3. "A background job is eating 85% CPU. You can't kill it. What do you do?"

**Answer:**
> *"First I'd find its PID and confirm its current priority:"*

```bash
ps -eo pid,user,ni,%cpu,comm | grep data-processor
# or
top -p $(pgrep data-processor)
```

> *"Then I'd use `renice` to reduce its priority to +10 or higher — it stays running but yields CPU to user-facing services:"*

```bash
renice -n 10 -p 5432
```

> *"I'd then verify the change took effect and monitor CPU usage to confirm user-facing services recovered. If it's still too greedy, I could push it to +19 — the lowest possible priority — without stopping it."*

---

### Q4. "Can a regular user run renice? Are there any restrictions?"

**Answer:**
> *"Yes, with restrictions. A regular user can only **increase** the nice value of their own processes — meaning they can make their process less important, but cannot give it more priority than it already has. Only root can decrease the nice value (raise priority) or change another user's process priority. The rule is: unprivileged users can only be 'nicer', never 'meaner'."*

```bash
# As devops user — works (making process nicer)
renice -n 10 -p 5432   ✓

# As devops user — FAILS (trying to raise priority)
renice -n -5 -p 5432   ✗  "Permission denied"

# As root — always works
sudo renice -n -5 -p 5432   ✓
```

---

### Q5. "How does the kernel actually use nice values?"

**Answer:**
> *"The Linux CFS — Completely Fair Scheduler — converts nice values into weights. Nice 0 has a weight of 1024. Each step up or down multiplies/divides by roughly 1.25. So nice +10 has a weight of 110, which means when competing against a nice-0 process, it gets only about 10% of the CPU. The key point is nice values are **relative** — the actual CPU% a process gets depends on what else is running and competing."*

---

### Q6. "What does the `N` flag mean in the STAT column in `ps` output?"

**Answer:**
> *"In `ps aux` output, the STAT column shows process state flags. `N` means the process has a **positive nice value** — it's been deprioritized below the default. So `RN` means the process is Running AND has a nice value greater than 0. Conversely, `<` in the STAT column means negative nice value — higher than normal priority. After running `renice -n 10`, you'd see the STAT change to include `N`."*

---

### Q7. "What's the difference between CPU priority (`nice`) and I/O priority (`ionice`)?"

**Answer:**
> *"They're completely independent. `nice`/`renice` controls how much **CPU time** a process gets from the scheduler. `ionice` controls how much **disk I/O bandwidth** a process gets. A backup job could be CPU-niced but still hammer the disk — you'd need both to fully throttle it. For the data-processing job that's CPU-heavy, `renice` is the right tool. If it were doing heavy disk reads/writes too, I'd combine: `nice -n 10 ionice -c 3 data-processor`."*

---

### Q8. "How would you make this priority change permanent?"

**Answer:**
> *"Three ways depending on the setup:"*

**Option 1 — systemd service (best for production):**
```ini
# /etc/systemd/system/data-processor.service
[Service]
Nice=10
ExecStart=/usr/bin/data-processor --batch
```

**Option 2 — limits.conf (per-user default):**
```bash
# /etc/security/limits.conf
devops  soft  priority  10
devops  hard  priority  10
```

**Option 3 — Wrapper script:**
```bash
#!/bin/bash
exec nice -n 10 /usr/bin/data-processor "$@"
```

> *"`renice` on a live PID is temporary — it resets if the process restarts. For a production service at RedHat, the systemd `Nice=` directive is the proper permanent solution."*

---

## Part 5: Cheat Sheet

```
NICE VALUE SCALE:
  -20 ←──────── 0 ────────→ +19
  highest     default     lowest
  priority               priority

KEY COMMANDS:
  ps -eo pid,user,ni,%cpu,comm     → see nice values
  renice -n 10 -p <PID>            → change running process
  nice -n 10 <command>             → launch with priority
  top → press 'r'                  → interactive renice

RULES:
  Regular user → can only INCREASE nice (less priority)
  Root only    → can DECREASE nice (more priority)
  Default nice = 0 for all processes

STAT COLUMN FLAGS:
  R = Running
  S = Sleeping
  N = Nice (positive nice value)  ← appears after renice +10
  < = High priority (negative nice value)

PERMANENT SOLUTIONS:
  systemd service → Nice=10
  limits.conf     → priority setting per user
```

> **RedHat interview tip:** They love when you explain **why** not just **how** — mention CFS, weight conversion, and the distinction between temporary `renice` vs permanent `systemd Nice=`. That's what separates a junior answer from a senior one.
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
5)
Company: TCS | Difficulty: Medium
Scenario
The developer account dev has been locked out of the server. Security logs indicate the SSH daemon's authentication failure limit was triggered.
Task
Check the logs to count exactly how many times the user failed `today`. Update the SSH configuration to increase the allowed login attempts `above that number`.
Example

```
root@server:~# tail /var/log/auth.log
Dec 23 09:12:01 server sshd[2201]: Failed password for dev from 10.0.0.5 port 5432 ssh2
Dec 23 09:12:05 server sshd[2201]: Failed password for dev from 10.0.0.5 port 5432 ssh2
Dec 23 09:12:08 server sshd[2201]: Failed password for dev from 10.0.0.5 port 5432 ssh2


```
## SSH Lockout Investigation — Full Deep Dive

---

## Part 1: Understand the Problem Simply

### What Happened?

```
Developer "dev" tries to SSH into server
          ↓
Types wrong password → once
Types wrong password → twice
Types wrong password → three times (or more)
          ↓
SSH daemon says: "Too many failures — BLOCKED"
          ↓
dev can no longer connect at all
even with correct password
```

### Where Does This Live in Linux?

```
/etc/ssh/sshd_config        ← SSH daemon config (MaxAuthTries lives here)
/var/log/auth.log           ← Authentication logs (Debian/Ubuntu)
/var/log/secure             ← Authentication logs (RHEL/CentOS/Rocky)
```

---

## Part 2: Let's Build and Execute It

### Step 1 — Check Today's Failed Login AttemptsEverything ran clean. Now let's understand every piece deeply.

---

## Part 2: Understand It Simply — The Story

### Why Does SSH Lock Out a User?

```
SSH has a built-in safety feature: MaxAuthTries

If someone fails to authenticate more times than MaxAuthTries allows
→ SSH kills that connection immediately
→ Any further attempts from that session are blocked

Default MaxAuthTries = 6
(Linux counts each auth method as one try — password, key, etc.)
```

### The Log File — What Each Part Means

```
May  3 09:12:01 server sshd[2201]: Failed password for dev from 10.0.0.5 port 5432 ssh2
  ↑               ↑      ↑    ↑           ↑           ↑         ↑           ↑        ↑
Date+Time      hostname daemon PID    what happened  user    source IP    port   protocol
```

### Why `grep "^$TODAY"` — Why Not Just grep for dev?

```
Log file has entries from MULTIPLE days:

Dec 22 08:10:01 ... Failed password for dev   ← yesterday
May  3 09:12:01 ... Failed password for dev   ← today ✓
May  3 09:12:05 ... Failed password for dev   ← today ✓

Without date filter:
  grep "Failed password for dev" | wc -l = 8  ← WRONG (includes old)

With date filter:
  grep "Failed password for dev" | grep "^May  3" | wc -l = 6  ← CORRECT
```

---

## Part 3: Every Command Explained

### The Log Grep Pipeline

```bash
grep "Failed password for dev" /var/log/auth.log \
| grep "^$TODAY" \
| wc -l
```

| Part | Does What |
|------|-----------|
| `grep "Failed password for dev"` | Find all login failures for user dev |
| `grep "^$TODAY"` | Keep only lines starting with today's date |
| `wc -l` | Count the matching lines |

```
^  means "start of line" in regex
$TODAY expands to "May  3" (note the double space for single-digit days)
```

---

### The `sed` Command for Config Update

```bash
sed -i "s/^#MaxAuthTries.*/MaxAuthTries 9/" /etc/ssh/sshd_config
```

| Part | Meaning |
|------|---------|
| `-i` | Edit the file **in-place** (no temp file needed) |
| `s/old/new/` | Substitute pattern |
| `^#MaxAuthTries` | Match lines starting with `#MaxAuthTries` (commented) |
| `.*` | Match everything after it |
| `MaxAuthTries 9` | Replace entire line with this |

```
Before:  #MaxAuthTries 6    ← commented out, default applies
After:   MaxAuthTries 9     ← active, overrides default
```

---

### Why `systemctl reload` Not `restart`?

```
systemctl restart sshd  →  KILLS all active SSH sessions ← DANGEROUS
                            Drops your own connection!

systemctl reload sshd   →  Rereads config gracefully
                            Existing connections STAY alive ✓
                            New connections use new config ✓
```

**Always use `reload` for sshd in production.**

---

### Auth Log Locations by Distro

```
Debian / Ubuntu:   /var/log/auth.log
RHEL / CentOS:     /var/log/secure
Rocky / AlmaLinux: /var/log/secure

# Universal alternative (works everywhere):
journalctl -u sshd --since today | grep "Failed password for dev"
```

---

## Part 4: Interview Questions — Full Detailed Answers

---

### Q1. "A developer is locked out of SSH. Walk me through how you'd fix it."

**Answer — say this step by step:**

> *"First I'd check the auth log to understand what triggered the lockout:"*

```bash
grep "Failed password for dev" /var/log/auth.log | grep "^$(date '+%b %e')"
```

> *"Then I'd count exactly how many failures happened today:"*

```bash
grep "Failed password for dev" /var/log/auth.log \
  | grep "^$(date '+%b %e')" | wc -l
```

> *"If the count equals or exceeds MaxAuthTries — that's the cause. I'd update `/etc/ssh/sshd_config` to set MaxAuthTries above that count, validate the config with `sshd -t`, then reload with `systemctl reload sshd` so existing sessions aren't dropped."*

> *"Finally I'd ask the developer to retry and confirm they can log in."*

---

### Q2. "What is `MaxAuthTries` in SSH? What's the default?"

**Answer:**
> *"`MaxAuthTries` in `/etc/ssh/sshd_config` sets the maximum number of authentication attempts allowed per SSH connection. The default is 6. Once a connection exceeds this limit, SSH terminates that session. Important nuance — it counts authentication **attempts** not just password tries. If you're using key-based auth, each key offered counts as one attempt. So on a machine with many keys in `~/.ssh/`, you might hit the limit before even trying a password."*

---

### Q3. "What's the difference between `/var/log/auth.log` and `journalctl`?"

**Answer:**
> *"`/var/log/auth.log` is a traditional flat text file managed by `rsyslog` or `syslog-ng`. It's human-readable and easy to grep. `journalctl` queries `systemd-journald`, which stores logs in a binary format with structured metadata — you can filter by unit, time range, priority, and more. On modern RHEL/Ubuntu systems both may coexist. For SSH specifically:"*

```bash
# Traditional log
grep "Failed password for dev" /var/log/auth.log

# journald — more powerful filtering
journalctl -u sshd --since today | grep "Failed password for dev"
journalctl -u sshd --since "2026-05-03 00:00:00" _UID=$(id -u dev)
```

> *"`journalctl` is better for time-range queries. grep on auth.log is faster for quick one-liners."*

---

### Q4. "Why use `systemctl reload` instead of `restart` for SSH?"

**Answer:**
> *"Because `restart` kills the sshd process and relaunches it — this terminates all active SSH sessions including your own. If you're connected remotely and run `systemctl restart sshd`, you get disconnected and potentially locked out. `reload` sends a SIGHUP signal to sshd, which makes it re-read its config file while keeping all existing connections alive. For SSH config changes, `reload` is always the safe choice."*

```bash
sudo sshd -t              # 1. Validate syntax first — no reload if broken
sudo systemctl reload sshd  # 2. Apply safely
sudo systemctl status sshd  # 3. Confirm still running
```

---

### Q5. "How do you make sure your `sshd_config` change doesn't break SSH?"

**Answer:**
> *"`sshd -t` is the safety net — it tests the config file for syntax errors without actually starting or reloading the daemon. If it returns exit code 0, the config is valid. I always run this before any reload. The workflow is:"*

```bash
# 1. Make the change
sed -i "s/^#MaxAuthTries.*/MaxAuthTries 9/" /etc/ssh/sshd_config

# 2. Test syntax — if this fails, DO NOT reload
sudo sshd -t
echo $?   # 0 = safe to proceed

# 3. Only reload if syntax is clean
sudo systemctl reload sshd
```

> *"Skipping `sshd -t` is how people accidentally break SSH and need console/out-of-band access to recover."*

---

### Q6. "Why filter by today's date in the log? Why not just count all failures?"

**Answer:**
> *"Because `MaxAuthTries` resets per connection — it's not cumulative across days. If I count all historical failures for dev, I might get 50, set MaxAuthTries to 53, but today's actual count was only 6. The SSH lockout only happened because today's session hit 6 attempts in one connection. Counting only today gives the real picture. I use `date '+%b %e'` to get the syslog-format date prefix like `May  3` and match only lines starting with that."*

---

### Q7. "What's the security implication of increasing `MaxAuthTries`?"

**Answer:**
> *"Increasing MaxAuthTries makes brute-force attacks slightly easier — more attempts per connection before getting cut off. So I wouldn't set it to 50 just because someone failed 6 times. My approach: set it just above the observed count with a small buffer — in this case 6 failures → MaxAuthTries 9. Then I'd also investigate WHY the developer failed 6 times — were they using a password manager that tried old passwords? Were they on a new device? The real fix might be setting up key-based auth and disabling password auth entirely with `PasswordAuthentication no`."*

---

### Q8. "What other SSH hardening would you apply at TCS/enterprise level?"

**Answer — show depth:**

```ini
# /etc/ssh/sshd_config — production hardened

MaxAuthTries 4               # tight limit
LoginGraceTime 30            # 30 sec to authenticate, not 2 min
PermitRootLogin no           # never allow root SSH
PasswordAuthentication no    # keys only
PubkeyAuthentication yes
AllowUsers dev devops admin  # whitelist specific users
ClientAliveInterval 300      # disconnect idle sessions after 5 min
ClientAliveCountMax 2
Banner /etc/ssh/banner.txt   # legal warning banner
```

> *"At enterprise level I'd also integrate with `fail2ban` to automatically ban IPs after N failures across connections, not just within one connection."*

---

## Part 5: Full Cheat Sheet

```
LOG FILES:
  /var/log/auth.log    → Debian/Ubuntu
  /var/log/secure      → RHEL/CentOS/Rocky

COUNT TODAY'S FAILURES:
  grep "Failed password for dev" /var/log/auth.log \
    | grep "^$(date '+%b %e')" | wc -l

UPDATE SSHD CONFIG:
  sed -i "s/^#MaxAuthTries.*/MaxAuthTries 9/" /etc/ssh/sshd_config

VALIDATE BEFORE RELOAD (CRITICAL):
  sudo sshd -t

RELOAD SAFELY (no disconnections):
  sudo systemctl reload sshd   ← USE THIS
  sudo systemctl restart sshd  ← NEVER in production

KEY SSHD CONFIG VALUES:
  MaxAuthTries 6     → max auth attempts per connection
  LoginGraceTime 2m  → time allowed to authenticate
  PermitRootLogin no → block root SSH
  AllowUsers dev     → whitelist users

JOURNALD ALTERNATIVE:
  journalctl -u sshd --since today | grep "Failed password for dev"
```

> **TCS interview tip:** They care about **process** — always mention `sshd -t` before reload, and bring up `fail2ban` or `AllowUsers` as a bonus. Shows you think about security holistically, not just the immediate fix.
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Detect Memory Leak by Monitoring RSS
Company: Google | Difficulty: Medium
Scenario
One of your long-running Node.Js services (process name `node`) has been slowing down over several hours of uptime. CPU usage is normal, disk I/O is normal, but the server is gradually running out of memory.
Task
Identify if any process is leaking memory and kill that process.


## Detect Memory Leak by Monitoring RSS — Full Deep Dive

---

## Part 1: Understand It Simply (The Story)

### What is a Memory Leak?

```
NORMAL process:                    LEAKING process:
  Allocates memory  ✓               Allocates memory  ✓
  Uses memory       ✓               Uses memory       ✓
  FREES memory      ✓               NEVER frees it    ✗
                                          ↓
                                    Memory keeps growing
                                    Server runs out of RAM
                                    Everything slows down/crashes
```

### What is RSS?

**RSS = Resident Set Size** = how much **real physical RAM** a process is currently using right now.

```
Process memory has multiple measurements:

VSZ  (Virtual Size)   = everything the process RESERVED
                        (includes shared libs, mapped files)
                        → can be huge but misleading

RSS  (Resident Size)  = what's actually IN physical RAM
                        → the REAL number you care about

SHR  (Shared Memory)  = portion shared with other processes
                        (shared libraries like libc)
```

Real world analogy:
```
VSZ = size of your apartment (total space available)
RSS = furniture actually inside it (space really used)
SHR = shared hallway/lobby (shared with neighbors)
```

### What Does a Memory Leak Look Like in RSS?

```
Healthy process RSS over time:
  100MB ──────────────────────────  (flat, stable)

Leaking process RSS over time:
  100MB ─╱
  150MB   ─╱
  200MB     ─╱
  300MB       ─╱                   (growing, never drops)
  500MB         ─╱                 (keeps climbing)
  SERVER OOM ↑  CRASH              (out of memory killer fires)
```

---

## Part 2: Build and Execute — Step by StepAll steps executed cleanly. Now let's understand every piece deeply.

---

## Part 3: Every Concept Explained Simply

### The `/proc` Filesystem — Your Real-Time Memory Window

```
/proc is NOT a real directory on disk
It's a virtual filesystem — the kernel exposes live
process data as readable files

/proc/531/
├── status      ← human-readable memory stats
├── smaps       ← every memory region with RSS breakdown
├── maps        ← memory map (what's mapped where)
└── fd/         ← open file descriptors
```

Reading memory without any tool:
```bash
cat /proc/531/status | grep -E "VmRSS|VmPeak|VmHWM"

VmPeak:   61280 kB   ← highest RSS this process ever hit
VmHWM:    55484 kB   ← High Water Mark (peak RSS)
VmRSS:    55484 kB   ← current RSS RIGHT NOW
VmSwap:       0 kB   ← how much is swapped out
```

**VmHWM keeps growing even if current RSS drops** — that's a strong leak signal.

---

### Why `smaps` is Better Than `status` for Leaks

```bash
cat /proc/PID/smaps

7f9d892ff000-7f9d8c200000 rw-p 00000000 00:00 0   ← anonymous heap
RSS:  47004 KB     ← 47MB sitting in this ONE region
```

The `rw-p 00000000 00:00 0` with no filename = **anonymous heap allocation** — exactly what a memory leak looks like. No file backing it, no shared library, just raw allocated memory that was never freed.

---

### The Kill Signals — Why Order Matters

```
SIGTERM (signal 15) → "Please shut down gracefully"
                       Process can catch this, cleanup, then exit
                       Node.js will flush buffers, close connections
                       Always try this FIRST

SIGKILL (signal 9)  → "Die. Now. No arguments."
                       Kernel kills it immediately
                       Process cannot catch or ignore this
                       Last resort — possible data corruption
```

```bash
# Always this order:
kill -15 $PID          # graceful — wait 5-10 seconds
kill -9  $PID          # force — only if -15 didn't work
```

---

### Memory Numbers Compared

```
ps aux output:
  VSZ    = 61280 KB  → 60MB virtual (reserved, not all in RAM)
  RSS    = 55484 KB  → 54MB actually in physical RAM ← use this

/proc/PID/status:
  VmSize = 61280 KB  → same as VSZ
  VmRSS  = 55484 KB  → same as RSS  ← most accurate live number
  VmHWM  = 55484 KB  → peak RSS ever → if this keeps climbing = leak

smaps total:
  Sum of all Rss fields → most accurate of all, line-level breakdown
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "What is RSS and why do you use it to detect memory leaks?"

**Answer:**
> *"RSS stands for Resident Set Size — it's the amount of physical RAM a process is actually using right now, not just what it has reserved. VSZ (Virtual Size) is misleading because it includes memory-mapped files, shared libraries, and reserved but unused space. RSS tells you what's really consuming RAM on the server. For detecting leaks, I monitor RSS over time — a healthy process has a stable RSS, a leaking process shows RSS that only goes up and never comes down, even during idle periods."*

```
VSZ = everything the process has reserved (often misleading)
RSS = what's actually in physical RAM (real impact)
```

---

### Q2. "Walk me through how you'd diagnose a Node.js process leaking memory."

**Answer — give this 4-step flow:**

> *"Step 1 — Confirm it's a memory problem, not CPU or I/O:"*
```bash
free -h                        # overall memory pressure
ps aux --sort=-%mem | head -10 # who's consuming most
```

> *"Step 2 — Identify the offender and check its RSS trend:"*
```bash
PID=$(pgrep -x node)
watch -n 2 "cat /proc/$PID/status | grep VmRSS"
# If VmRSS grows every few seconds → leak confirmed
```

> *"Step 3 — Inspect memory regions with smaps to confirm heap growth:"*
```bash
awk '/^[0-9a-f]/{r=$0} /^Rss/{print $2, r}' /proc/$PID/smaps \
  | sort -rn | head -5
# Anonymous heap regions with large RSS = leaked memory
```

> *"Step 4 — Kill gracefully, force if needed:"*
```bash
kill -15 $PID    # SIGTERM first
sleep 5
kill -9  $PID    # SIGKILL if still alive
```

---

### Q3. "What's the difference between SIGTERM and SIGKILL? Which do you use first?"

**Answer:**
> *"SIGTERM (15) is a polite request to terminate — the process can catch it, run cleanup code, flush logs, close database connections, then exit. Node.js handles SIGTERM and will gracefully drain active requests. SIGKILL (9) is unconditional — the kernel destroys the process immediately, no cleanup possible, which can leave files half-written or database transactions incomplete. I always try SIGTERM first and wait 5–10 seconds. Only if the process is frozen or ignoring SIGTERM do I escalate to SIGKILL."*

---

### Q4. "What is `/proc/PID/smaps` and what does it tell you about leaks?"

**Answer:**
> *"`/proc/PID/smaps` breaks down every single memory region a process has mapped — including the RSS for each region individually. For a memory leak, the smoking gun is a large anonymous heap region — shown as `rw-p 00000000 00:00 0` with no filename — that keeps growing. Named regions like `/usr/lib/libc.so` are shared libraries and are expected. But 47MB of anonymous heap with no file backing it means the process allocated that memory at runtime and never freed it."*

```
Anonymous region (leak):     7f9d892ff000 rw-p 00000000 00:00 0
Named region (normal):       7f9d8c3b0000 r--p libc.so.6
```

---

### Q5. "What is VmHWM in `/proc/PID/status` and why is it useful?"

**Answer:**
> *"VmHWM is Virtual Memory High Water Mark — it records the peak RSS the process has ever reached since it started. It's useful for leak detection because even if the process momentarily frees some memory (RSS drops), VmHWM never decreases. If I come back 6 hours later and VmHWM is 800MB but the process started at 100MB, that's strong evidence of a historical leak even if current RSS looks lower. It's the process's memory high score."*

```bash
cat /proc/$PID/status | grep VmHWM
VmHWM:   819200 kB    # peaked at 800MB — leak happened
```

---

### Q6. "How do you find a leaking process if you don't know the process name?"

**Answer:**
> *"Several approaches. First I'd look at overall memory pressure:"*

```bash
free -h              # how much RAM is left

# Sort all processes by RSS descending
ps aux --sort=-%mem | head -15

# Or use smem for more accurate PSS (Proportional Set Size)
smem -r | head -10
```

> *"PSS (Proportional Set Size) from `smem` is actually more accurate than RSS for systems with many shared libraries — it counts shared memory proportionally. But for a quick investigation, `ps --sort=-%mem` is fast and tells you who the biggest consumers are within seconds."*

---

### Q7. "What is the OOM Killer and when does it fire?"

**Answer:**
> *"The OOM Killer — Out of Memory Killer — is the Linux kernel's last resort when physical RAM and swap are both exhausted. It scores all running processes based on memory usage, bad behavior history, and OOM score adjustments, then kills the process with the highest score. For a Node.js leak, the OOM Killer might kill it automatically — but usually kills the wrong process first. You can see OOM kills in dmesg:"*

```bash
dmesg | grep -i "oom\|killed process"
# Out of memory: Kill process 5432 (node) score 892 or sacrifice child
```

> *"To protect critical processes from OOM kill, set a negative `oom_score_adj`:"*

```bash
# Protect postgres from OOM killer
echo -1000 > /proc/$(pgrep postgres)/oom_score_adj

# Make leaking process OOM-killable first
echo 1000 > /proc/$LEAK_PID/oom_score_adj
```

---

### Q8. "How would you permanently prevent this Node.js memory leak from crashing the server?"

**Answer — give 3 layers:**

> *"Layer 1 — Process manager with memory limits (immediate fix):"*
```bash
# PM2 (standard Node.js process manager)
pm2 start app.js --max-memory-restart 500M
# Automatically restarts if RSS exceeds 500MB
```

> *"Layer 2 — cgroup memory limits (OS-level hard cap):"*
```bash
# systemd service with memory limit
# /etc/systemd/system/node-app.service
[Service]
MemoryMax=512M          # hard kill at 512MB
MemoryHigh=400M         # throttle at 400MB (soft limit)
Restart=always          # auto-restart if killed
```

> *"Layer 3 — Alerting before it becomes critical:"*
```bash
# Alert when node RSS exceeds 400MB
RSS=$(cat /proc/$(pgrep node)/status | awk '/VmRSS/{print $2}')
[ $RSS -gt 409600 ] && echo "ALERT: node RSS ${RSS}KB" | mail -s "Memory leak" ops@company.com
```

> *"The real fix is code-level — finding the unclosed event listener, circular reference, or growing cache in the Node.js app using `--inspect` with Chrome DevTools heap snapshots. But the above buys time and prevents outages while the dev team patches it."*

---

## Part 5: Full Cheat Sheet

```
MEMORY METRICS:
  VSZ  = Virtual Size (reserved, misleading)
  RSS  = Resident Set Size (real RAM used) ← use this
  SHR  = Shared memory (libs shared between processes)
  PSS  = Proportional Set Size (most accurate, needs smem)

KEY FILES IN /proc:
  /proc/PID/status    → VmRSS, VmHWM, VmPeak
  /proc/PID/smaps     → per-region RSS breakdown
  /proc/PID/maps      → what's mapped where

FIND THE LEAKER:
  ps aux --sort=-%mem | head -10     # quick sort by RSS
  watch -n 2 "ps aux --sort=-%mem | head -5"  # live watch

MONITOR RSS GROWTH:
  watch -n 3 "cat /proc/PID/status | grep VmRSS"

KILL ORDER (always):
  kill -15 $PID   →  SIGTERM (graceful, try first)
  kill -9  $PID   →  SIGKILL (force, last resort)

LEAK SIGNALS:
  VmRSS grows over time          ← leak in progress
  VmHWM >> starting RSS          ← leaked historically
  Large anonymous smaps region   ← heap never freed
  Swap usage appearing           ← RAM exhausted, spilling

PREVENT RECURRENCE:
  PM2:      --max-memory-restart 500M
  systemd:  MemoryMax=512M
  cgroup:   memory.limit_in_bytes
```

> **Google interview tip:** They'll push you on **VmHWM vs VmRSS**, the difference between **RSS and PSS**, and why SIGTERM before SIGKILL. Mention the OOM killer and `oom_score_adj` — it shows you understand what happens when monitoring fails and the kernel has to intervene. That's senior-level depth they're looking for.

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Discover Unexpected Background Jobs
Company: Plus500 | Difficulty: Medium
Scenario
You have noticed an unexpected spike in system load. You suspect a batch of recently spawned jobs is responsible and need to isolate processes that started within the last few minutes.
Task
Identify all processes started within the last 10 minutes and save their PID, User, Start Date (`lstart`), and Command to `/home/devops/recent_processes.txt`. Once recent processes written to the file Terminate the Suspicious processes (i.e any process that doesn't belong to the root user or systemd).
Example
The file `/home/devops/recent_processes.txt` should contain the list of recently started processes:

```
  PID USER     STARTED                     CMD
 8234 deploy   Mon Oct 29 16:37:12 2025    /opt/scripts/deploy.sh
 8235 deploy   Mon Oct 29 16:37:15 2025    bash ./worker_start.sh

```
## Discover Unexpected Background Jobs — Full Deep Dive

---

## Part 1: Understand the Problem Simply

### What's Happening?

```
Normal server load:          Spike detected:
  CPU  ██░░░░░░  20%           CPU  ████████  85%
  Load  0.5                    Load  8.2  ← something spawned
  RAM  ███░░░░░  40%           RAM  ██████░░  75%

Question: What started in the last 10 minutes?
```

### How Linux Tracks Process Start Time

Every process in Linux has a **start time** recorded by the kernel the moment it's created. You can read it from:

```
/proc/PID/stat         ← raw clock ticks since boot
ps -o lstart           ← human-readable full timestamp
                          (lstart = Long Start time)
```

```
Regular start time:   "14:35"        → only shows time (ambiguous)
lstart:               "Mon May 3 14:35:22 2026"  → full date + time ✓
```

---

## Part 2: Build and Execute — Step by StepAll suspicious processes killed. The `<defunct>` entries are zombie processes — perfect teaching moment for the explanation. Now let's understand everything deeply.

---

## Part 3: Every Concept Explained Simply

### How `lstart` Works vs Regular Time

```bash
ps -o start    # SHORT format — shows only time if today
               # 13:19  ← ambiguous, which day?

ps -o lstart   # LONG format — full timestamp always
               # Sun May  3 13:19:28 2026  ← exact, unambiguous
```

Why lstart matters for this task:
```
If server has been running for days, logs show:
  PID 100  start=09:15   ← was this today? yesterday? 3 days ago?
  PID 200  start=09:15   ← same time, different day entirely?

lstart removes ALL ambiguity:
  PID 100  Sun May  3 09:15:00 2026   ← exact date + time
  PID 200  Sat May  2 09:15:00 2026   ← clearly yesterday
```

---

### The Time Comparison Logic — Core of the Script

```bash
CUTOFF=$(date -d "10 minutes ago" +%s)
# date -d "10 minutes ago"  → human time → -d parses it
# +%s                       → convert to EPOCH (seconds since Jan 1 1970)
# Result: 1746274481        ← single number, easy to compare

PROC_EPOCH=$(date -d "$LSTART" +%s)
# Convert each process's lstart to epoch too

if [ "$PROC_EPOCH" -ge "$CUTOFF" ]; then
    # Process started after cutoff = recent = suspicious
fi
```

Why epoch (seconds)?
```
Cannot compare "Sun May 3 13:19:28" > "Sun May 3 13:14:41" directly in bash
CAN compare  1746274768 > 1746274481  — just two integers ✓
```

---

### Zombie Processes — What Are Those `<defunct>`?

After killing, you saw:
```
deploy  5579  [bash] <defunct>
deploy  8075  [sleep] <defunct>
```

```
ZOMBIE = process is dead (we killed it) BUT
         its parent process hasn't called wait()
         to collect its exit status yet

The entry stays in the process table briefly
Taking up a PID slot but ZERO memory, ZERO CPU

How it gets cleaned up:
  Parent process calls wait() → zombie disappears ✓
  Parent dies too             → init/systemd adopts + cleans up ✓
  
Zombies are harmless UNLESS there are thousands of them
(PID table exhaustion)
```

---

### The Kill Strategy — SIGTERM Then SIGKILL

```
SIGTERM (15) sent first:
    Process wakes up → runs cleanup code → exits cleanly
    Good for: Node.js, databases, apps with open connections

Wait 0.5–5 seconds

Still alive? → SIGKILL (9):
    Kernel destroys it instantly, no cleanup
    Good for: frozen/unresponsive processes

Why sleep 400 needed SIGKILL:
    sleep ignores SIGTERM in some implementations
    → kernel SIGKILL was required
```

---

### What `/proc` Tells You About Process Start Time

```bash
# Alternative to ps lstart — reading directly from kernel
cat /proc/6542/stat | awk '{print $22}'
# Output: 18478234  ← clock ticks since system boot

# Convert to human time:
BOOT=$(date -d "$(uptime -s)" +%s)
TICKS=18478234
HZ=100  # clock ticks per second (getconf CLK_TCK)
START=$((BOOT + TICKS / HZ))
date -d "@$START"
```

`ps lstart` does this math for you — but knowing the `/proc` source matters in interviews.

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How do you find processes started in the last 10 minutes?"

**Answer:**
> *"I use `ps` with the `lstart` format option — it gives the full date and time a process started, unlike the short `start` field which is ambiguous. Then I convert both the cutoff time and each process's lstart to epoch seconds using `date +%s`, which lets me do simple integer comparison in bash."*

```bash
CUTOFF=$(date -d "10 minutes ago" +%s)

ps -eo pid,user,lstart,cmd --no-headers | while read PID USER D1 D2 D3 D4 D5 CMD; do
    LSTART="$D1 $D2 $D3 $D4 $D5"
    PROC_EPOCH=$(date -d "$LSTART" +%s 2>/dev/null)
    [ "$PROC_EPOCH" -ge "$CUTOFF" ] && echo "$PID $USER $CMD"
done
```

---

### Q2. "What's the difference between `lstart` and `start` in ps output?"

**Answer:**
> *"`start` shows a short format — if the process started today, it shows just the time like `13:19`. If it started earlier, it might show the date without the year. It's ambiguous on long-running servers. `lstart` always shows the full timestamp — day, month, date, time, and year — so there's zero ambiguity regardless of when the process started. For time-based comparisons in scripts, `lstart` is the only reliable option."*

```bash
ps -o start    # 13:19       ← today only, no date
ps -o lstart   # Sun May  3 13:19:28 2026  ← always complete
```

---

### Q3. "What is epoch time and why use it for time comparison in bash?"

**Answer:**
> *"Epoch time (Unix timestamp) is the number of seconds since January 1, 1970. I use it in bash because you can't directly compare timestamp strings like `'May 3 13:19'` with `>` or `<` operators — bash doesn't know how to parse them. But epoch is just an integer — comparing two integers is trivial. `date +%s` converts any human-readable time to epoch, and then the comparison becomes simple arithmetic."*

```bash
date -d "10 minutes ago" +%s   # → 1746274481
date -d "Sun May 3 13:19:00 2026" +%s  # → 1746274740

# Now compare:
[ 1746274740 -ge 1746274481 ] && echo "Recent"   # ✓
```

---

### Q4. "How do you decide which processes are suspicious?"

**Answer:**
> *"Context-dependent, but the baseline rule is: processes not owned by `root` or `systemd` that appeared suddenly during a load spike are suspicious. In production I'd also look at: unusual command paths like `/tmp` or `/dev/shm` which are commonly used by malware, processes with no associated service, anonymous processes or renamed processes, and processes spawned from unexpected parent PIDs. For this task, any non-root, recently-started process gets terminated."*

```bash
# Red flags to check:
ls -la /proc/$PID/exe        # where is the binary? /tmp = suspicious
cat /proc/$PID/cmdline       # what arguments?
ls -la /proc/$PID/fd         # open file descriptors — network connections?
```

---

### Q5. "What is a zombie process and should you kill it?"

**Answer:**
> *"A zombie process is a process that has finished executing but still has an entry in the process table because its parent hasn't called `wait()` to read its exit status. It consumes no CPU or memory — just a PID slot. You can't kill a zombie with SIGKILL because it's already dead. The fix is to kill or fix its parent process, which then triggers cleanup. Thousands of zombies are a problem — they can exhaust the PID table — but a few are normal and harmless."*

```bash
# Spot zombies:
ps aux | awk '$8 == "Z"'
# or
ps aux | grep defunct

# Fix: kill the parent
kill -9 $(ps -o ppid= -p <zombie_pid>)
```

---

### Q6. "Why SIGTERM before SIGKILL? When would you skip straight to SIGKILL?"

**Answer:**
> *"SIGTERM gives the process a chance to clean up — flush buffers, close database connections, finish in-flight requests. Skipping it risks data corruption or broken connections. I'd go straight to SIGKILL only if: the process is completely frozen and not responding to any signals, it's a confirmed malicious process where cleanup doesn't matter, or it's a simple stateless process like `sleep` where there's nothing to clean up. In security incidents, I might use SIGKILL directly to prevent the process from doing any further damage during 'cleanup'."*

---

### Q7. "How would you find processes spawned from a suspicious parent?"

**Answer:**
> *"I'd use `pstree` or query PPID (Parent PID) in `/proc`. If a suspicious process spawned children, I want to kill the whole tree."*

```bash
# See full process tree
pstree -p $SUSPICIOUS_PID

# Find all children of a PID
pgrep -P $PPID

# Kill entire process group
kill -9 -$PGID   # negative PID = kill process group

# Or use pkill on the tree
pkill -9 -P $SUSPICIOUS_PID   # kill all children
kill -9 $SUSPICIOUS_PID       # then kill parent
```

---

### Q8. "How would you make this detection automated and production-ready?"

**Answer:**
> *"Three layers: detection, alerting, and response."*

**Detection — cron job every 5 minutes:**
```bash
# /etc/cron.d/suspicious_process_monitor
*/5 * * * * root /usr/local/bin/detect_recent_processes.sh
```

**Script with alerting:**
```bash
#!/bin/bash
CUTOFF=$(date -d "5 minutes ago" +%s)
SUSPICIOUS=""

while read PID USER D1 D2 D3 D4 D5 CMD; do
    LSTART="$D1 $D2 $D3 $D4 $D5"
    EPOCH=$(date -d "$LSTART" +%s 2>/dev/null) || continue
    [ "$EPOCH" -lt "$CUTOFF" ] && continue
    [ "$USER" = "root" ] && continue
    SUSPICIOUS="$SUSPICIOUS\n$PID $USER $CMD"
done < <(ps -eo pid,user,lstart,cmd --no-headers)

if [ -n "$SUSPICIOUS" ]; then
    echo -e "Suspicious processes:\n$SUSPICIOUS" \
      | mail -s "ALERT: Unexpected processes on $(hostname)" ops@company.com
fi
```

> *"In enterprise setups at Plus500-level, I'd integrate with auditd for process exec logging, and SIEM tools like Splunk or ELK to correlate process spawning with login events and network connections."*

---

## Part 5: Cheat Sheet

```
KEY COMMANDS:
  ps -eo pid,user,lstart,cmd    → all processes with full start time
  date -d "10 minutes ago" +%s  → epoch cutoff timestamp
  date -d "$LSTART" +%s         → convert lstart to epoch
  kill -15 $PID                 → SIGTERM (graceful, try first)
  kill -9  $PID                 → SIGKILL (force, last resort)
  kill -9 -$PGID                → kill entire process group
  pgrep -P $PPID                → find children of a process

ZOMBIE FACTS:
  State = Z in ps STAT column
  Shows as <defunct> in CMD
  No CPU, no memory — just a PID slot
  Cannot be killed (already dead)
  Fix = kill or wait on parent process

SUSPICIOUS PROCESS SIGNALS:
  Non-root user                 → unexpected user account
  Path in /tmp or /dev/shm      → malware hiding spot
  No parent service             → not started by systemd
  Random-looking name           → possible obfuscation
  lstart during load spike      → timing correlation

REPORT FILE FORMAT:
  printf "%-6s %-12s %-28s %s\n" PID USER STARTED CMD
```

> **Plus500 interview tip:** They operate in fintech where **audit trails matter** as much as the fix. Always mention saving findings to a file before killing anything — log first, act second. Also mention `auditd` for persistent process exec logging — that's the production-grade answer they're looking for.
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
# Fix Inode Exhaustion Issue
> **Company:** DeutscheBank | **Difficulty:** Medium
---
#### Scenario:
Your server cannot create new files. Commands like `touch` fail with "No space left on device" errors, but `df -h` shows plenty of free disk space. The filesystem has exhausted available inodes.
#### Task:
Save inode usage to `/home/interview/inode_usage.txt`, find which directory contains excessive files, save the problematic directory path to `/home/interview/problem_directory.txt`, clean up the files, and verify the fix.
#### **Example**
**File: /home/interview/inode_usage.txt**
```
Filesystem            Inodes   IUsed   IFree IUse% Mounted on
/dev/sda1            6553600 6553600       0  100% /
/dev/sdb1            3276800  125000 3151800    4% /data
```
**File: /home/interview/problem_directory.txt**
```
/var/spool/postfix/maildrop
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/fix-inode-exhaustion-issue)

## Fix Inode Exhaustion — Full Deep Dive

---

## Part 1: Understand It Simply (The Story)

### What is an Inode?

Every file on Linux has **two parts:**

```
┌─────────────────────────────────────────────┐
│  INODE (metadata)      │  DATA BLOCKS        │
│  ─────────────────      │  ────────────        │
│  • File size            │  actual content     │
│  • Owner / permissions  │  "Hello World"      │
│  • Timestamps           │                     │
│  • Pointers to data     │                     │
│  • NOT the filename     │                     │
└─────────────────────────────────────────────┘
         ↑
  Fixed number created
  when filesystem is formatted
  (you can't add more without reformatting)
```

### Why Does Inode Exhaustion Happen?

```
Filesystem formatted with: 6,553,600 inodes
                                ↓
Each file created = 1 inode consumed
                                ↓
Someone creates millions of tiny files
(email queues, temp files, cache, logs)
                                ↓
All 6,553,600 inodes used up
                                ↓
touch newfile.txt → "No space left on device" ✗
                                ↓
df -h shows: 200GB free ← DISK has space
df -i shows: 0 free     ← INODES exhausted
```

### The Analogy

```
Disk space  = the physical shelves in a warehouse
Inodes      = the label slots in the catalog system

You can have empty shelves (disk space free)
but if your catalog is full (no inode slots)
you cannot register any new item — period.
```

---

## Part 2: Build and Execute

### Simulate the Problem FirstAll steps executed perfectly. Now let's understand everything deeply.

---

## Part 3: Every Concept Explained Simply

### The Inode Architecture

```
When you format a disk (mkfs.ext4):
─────────────────────────────────────

  mkfs.ext4 -N 6553600 /dev/sda1
                ↑
        inode count fixed HERE — forever
        (default: 1 inode per 16KB of disk)

Two separate pools created:
┌─────────────────────┐  ┌─────────────────────┐
│   DATA BLOCKS       │  │   INODE TABLE       │
│   (disk space)      │  │   (file metadata)   │
│                     │  │                     │
│   200GB free        │  │   6,553,600 slots   │
│   tracked by df -h  │  │   tracked by df -i  │
└─────────────────────┘  └─────────────────────┘

Both pools are INDEPENDENT — one can fill while other is empty
```

### What Each `df` Flag Shows

```bash
df -h     # human readable DISK SPACE
Filesystem   Size  Used  Avail  Use%
/dev/sda1    200G   50G   150G   25%   ← plenty of room

df -i     # inode usage
Filesystem   Inodes   IUsed   IFree  IUse%
/dev/sda1   6553600  6553600      0   100%  ← EXHAUSTED ← real problem
```

```
The sysadmin trap:
  Server appears to have 150GB free
  touch file.txt → "No space left on device"
  Panics → buys new disk → doesn't fix it
  (Disk space wasn't the issue — inodes were)
```

---

### Why `find + wc -l` Not `du -sh`?

```bash
du -sh /var/spool/postfix/maildrop
# Output: 244K   ← tiny! wouldn't alert you at all

find /var/spool/postfix/maildrop -type f | wc -l
# Output: 8000   ← 8000 files! THIS is the problem
```

```
du  = measures DISK BYTES used
find | wc -l = counts FILE COUNT = inode consumption

For inode exhaustion, FILE COUNT is what matters
not disk bytes — these are orthogonal metrics
```

---

### The Three Cleanup Methods — When to Use Each

```bash
# METHOD 1: find -delete (BEST — handles any file count)
find /path -type f -delete
# Works even with 10 million files
# Doesn't hit shell argument limits
# Handles filenames with spaces/special chars

# METHOD 2: rm with glob (FAILS on huge counts)
rm /path/*
# bash: /bin/rm: Argument list too long  ← ERROR at ~100K files
# /proc/sys/kernel/arg-max limits this

# METHOD 3: rsync empty dir trick (nuclear option)
mkdir /tmp/empty
rsync -a --delete /tmp/empty/ /path/
# Replaces entire directory with empty one
# Fastest for millions of files
# Use when find -delete is too slow
```

```
Rule of thumb:
< 100K files  → rm works fine
100K - 1M     → find -delete
> 1M files    → rsync empty dir trick
```

---

### Common Real-World Causes of Inode Exhaustion

```
1. Mail queue flood (/var/spool/postfix/maildrop)
   → Broken mail server queuing but not sending
   → Fix: flush queue + fix mail config

2. PHP session files (/var/lib/php/sessions)
   → Each web visitor creates a session file
   → Fix: session cleanup cron + shorter TTL

3. Systemd journal (/run/log/journal)
   → Each log entry can create files
   → Fix: journalctl --vacuum-files=100

4. Temporary files (/tmp, /var/tmp)
   → Apps that create temp files without cleanup
   → Fix: tmpwatch/systemd-tmpfiles

5. Container image layers
   → Docker images with many small files
   → Fix: docker system prune
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "Server says 'No space left on device' but `df -h` shows free space. What's wrong?"

**Answer — The perfect answer:**
> *"Classic inode exhaustion. `df -h` shows disk block usage — how many bytes are used. But `df -i` shows inode usage — how many files exist. These are two completely separate resource pools created when the filesystem is formatted. A filesystem can have gigabytes of free disk space but zero available inodes if too many files were created. I'd immediately run `df -i` to confirm, then hunt for the directory with the most files using `find` piped to `wc -l`."*

```bash
df -i                                     # confirm inode exhaustion
find / -maxdepth 4 -type d | while read dir; do
    COUNT=$(find "$dir" -maxdepth 1 -type f 2>/dev/null | wc -l)
    echo "$COUNT $dir"
done | sort -rn | head -10               # find the offender
```

---

### Q2. "What is an inode and what information does it store?"

**Answer:**
> *"An inode is a data structure that stores metadata about a file. It contains: file size, owner UID/GID, permissions, timestamps (atime, mtime, ctime), hard link count, and pointers to the data blocks where the file's content lives. Crucially, it does NOT store the filename — that mapping lives in the directory entry. Every file and directory has exactly one inode. The total number of inodes is fixed at filesystem creation time with `mkfs` and cannot be changed without reformatting."*

```bash
# View a file's inode number and metadata
stat /etc/passwd
ls -i /etc/passwd     # shows inode number
```

---

### Q3. "Why does `rm /path/*` fail with 'Argument list too long'?"

**Answer:**
> *"When you run `rm /path/*`, the shell expands `*` into a list of all filenames and passes them as arguments to `rm`. Linux has a kernel limit on argument list size — defined in `/proc/sys/kernel/arg-max`, typically around 2MB. With hundreds of thousands of files, the expanded argument list exceeds this limit. The fix is `find -delete` which doesn't use shell argument expansion — it handles each file internally. For millions of files, the `rsync` empty directory trick is even faster."*

```bash
cat /proc/sys/kernel/arg_max    # check limit (~2MB default)

# These FAIL at ~100K files:
rm /path/*
ls /path/ | xargs rm

# These WORK at any count:
find /path -type f -delete       # reliable
rsync -a --delete /empty/ /path/ # fastest
```

---

### Q4. "How do you find which directory is consuming the most inodes?"

**Answer:**
> *"I use `find` to count files per directory — NOT `du`, because `du` measures bytes, not file count. The key insight is that inode exhaustion is about file count, not disk size. I scan common problem directories first — `/var/spool`, `/tmp`, `/var/lib/php` — then drill deeper into the worst offender."*

```bash
# Quick scan of common offenders
for dir in /var/spool /tmp /var/lib /var/cache; do
    COUNT=$(find "$dir" -type f 2>/dev/null | wc -l)
    echo "$COUNT $dir"
done | sort -rn

# Deep scan from root (slower but thorough)
find / -xdev -type d 2>/dev/null | while read dir; do
    COUNT=$(find "$dir" -maxdepth 1 -type f 2>/dev/null | wc -l)
    [ "$COUNT" -gt 1000 ] && echo "$COUNT $dir"
done | sort -rn
```

> *"`-xdev` flag is important — it prevents `find` from crossing filesystem boundaries, so you don't accidentally scan mounted NFS shares or other filesystems."*

---

### Q5. "How do you increase inodes without reformatting? Can you?"

**Answer:**
> *"On `ext4` — no, you cannot add inodes to an existing filesystem without reformatting. The inode table is baked in at `mkfs` time. However, there are workarounds:"*

```bash
# Option 1: Create new filesystem with more inodes
# bytes-per-inode = 4096 means 1 inode per 4KB (4x more inodes)
mkfs.ext4 -i 4096 /dev/sdb1   # default is -i 16384

# Option 2: XFS filesystem — inodes grow dynamically
# XFS allocates inodes on demand, no fixed limit
# Great for filesystems expected to have many small files

# Option 3: tmpfs for temp files — inodes backed by RAM
mount -t tmpfs -o size=1G,nr_inodes=2000000 tmpfs /var/tmp

# Option 4: Immediate relief without reformat
# Move the problem directory to a different filesystem
# that has free inodes and bind-mount it back
mount --bind /mnt/extra/maildrop /var/spool/postfix/maildrop
```

> *"For production systems at DeutscheBank scale, XFS is preferred for data directories specifically because it avoids this problem entirely."*

---

### Q6. "What are some permanent fixes to prevent inode exhaustion recurring?"

**Answer — give 3 layers:**

> **Layer 1 — Cleanup automation:**
```bash
# Cron job to clean stale temp files
# /etc/cron.daily/clean-maildrop
find /var/spool/postfix/maildrop -type f -mtime +1 -delete

# systemd-tmpfiles for /tmp (already built-in on most systems)
# /etc/tmpfiles.d/cleanup.conf
d /var/tmp 1777 root root 7d
```

> **Layer 2 — Monitor before it hits 100%:**
```bash
# Alert at 80% inode usage
IUSE=$(df -i / | awk 'NR==2 {print $5}' | tr -d '%')
[ "$IUSE" -gt 80 ] && echo "ALERT: Inodes at ${IUSE}%" | mail ops@bank.com
```

> **Layer 3 — Fix root cause:**
```bash
# For postfix: fix mail delivery so queue drains
postqueue -p    # view queue
postqueue -f    # flush queue
postsuper -d ALL  # delete all queued messages (nuclear)
```

---

### Q7. "What's the difference between hard links and symlinks in relation to inodes?"

**Answer:**
> *"A hard link is a directory entry that points to an existing inode — it does NOT create a new inode. Two hard links to the same file share one inode, so creating a hard link doesn't consume an additional inode. A symlink (soft link) IS a separate file with its own inode, containing the path to the target. This matters for inode exhaustion — if someone is creating millions of symlinks, each one consumes an inode. Hard links wouldn't cause the same problem. You can see the hard link count in `ls -l` — the number after permissions."*

```bash
ls -li file.txt             # shows inode number
ln file.txt hardlink.txt    # same inode number, count goes to 2
ln -s file.txt symlink.txt  # new inode for the symlink itself

stat file.txt | grep Inode  # shows inode + link count
```

---

## Part 5: Cheat Sheet

```
DIAGNOSIS:
  df -h           → disk space (bytes)
  df -i           → inode usage ← check this first
  df -i /         → just root filesystem

FIND THE OFFENDER:
  find /var -type f | wc -l          → count all files
  find /var -maxdepth 1 -type d |    → per subdirectory
    xargs -I{} sh -c 'echo "$(find {} -type f | wc -l) {}"' |
    sort -rn | head -10

CLEANUP (by file count):
  < 100K     →  rm /path/*
  < 1M       →  find /path -type f -delete
  > 1M       →  rsync -a --delete /empty/ /path/

KEY FLAGS:
  find -xdev     → don't cross filesystem boundaries
  find -maxdepth → limit search depth (faster)
  find -type f   → files only (not directories)
  df -i          → inode stats

COMMON CULPRITS:
  /var/spool/postfix/maildrop  → mail queue flood
  /var/lib/php/sessions        → web session files
  /tmp and /var/tmp            → uncleaned temp files
  /var/log                     → log files (if unrotated)
  ~/.thumbnails                → desktop thumbnail cache

PREVENTION:
  mkfs.ext4 -i 4096  → more inodes at format time
  Use XFS             → dynamic inode allocation
  Monitor: df -i in cron + alert at 80%
```

> **DeutscheBank interview tip:** Finance companies hate unplanned outages. Always mention **monitoring before it hits 100%** with an 80% alert threshold, and explain **root cause analysis** — finding WHY the files accumulated (broken mail daemon, unrotated logs) not just cleaning up the symptom. That's what separates a reactive sysadmin from a proactive SRE.
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Handling Large Log Archives
> **Company:** Amazon | **Difficulty:** Easy
---
#### **Scenario**
During an incident investigation, you pulled a massive log export from `/var/log/app/access.log` that's several gigabytes in size. Your analysis tools and editors can't handle the entire file at once.
#### **Task**
You need to split it into smaller, more manageable chunks for parallel review. Create a directory `/tmp/log_parts/` to store the split files, split `/var/log/app/access.log` into smaller files containing 100 lines each, name the output files sequentially with the prefix `access_part_` (e.g., `access_part_aa`, `access_part_ab`, etc.), ensure the original log file remains untouched.
#### **Example**
```
# Before (single large log file)
/var/log/app/access.log: 375 lines, 2.5 GB
Cannot be opened by standard analysis tools
```
```
# After (log split into manageable chunks)
100 /tmp/log_parts/access_part_aa
100 /tmp/log_parts/access_part_ab
100 /tmp/log_parts/access_part_ac
 75 /tmp/log_parts/access_part_ad
375 total
Original file intact, 4 parts created for parallel analysis
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/handling-large-log-archives)

## Handling Large Log Archives — Full Deep Dive

---

## Part 1: Understand It Simply (The Story)

### Why Split Large Files?

```
Problem: access.log = 2.5 GB
         ↓
  vim access.log      → editor crashes / hangs
  grep "ERROR" (2.5GB)→ slow, serial, one CPU
  cat access.log      → terminal floods for minutes
  email to teammate   → attachment too large
         ↓
Solution: Split into 100-line chunks
         ↓
  access_part_aa  → 100 lines  → analyst 1
  access_part_ab  → 100 lines  → analyst 2
  access_part_ac  → 100 lines  → analyst 3
  access_part_ad  →  75 lines  → analyst 4
         ↓
  Parallel review — 4x faster investigation
```

### How `split` Names Files

```
split uses alphabetical suffix pairs:

aa → ab → ac → ... → az →
ba → bb → bc → ... → bz →
ca → ...

That gives you: 26 × 26 = 676 possible chunks
For 100-line chunks: handles files up to 67,600 lines
Need more? Use -a 3 for triple suffix (aa → zzz = 17,576 chunks)
```

---

## Part 2: Build and Execute

### Step 1 — Create the Log File### Step 2 — Record Original File State (Safety Check)### Step 3 — Create Output Directory and Split### Step 4 — Verify Line Counts Match the Example Exactly### Step 5 — Verify Original File is Untouched### Step 6 — Bonus: Reconstruct and ValidateOutput matches the expected format exactly. Now the deep explanation.

---

## Part 3: Every Concept Explained Simply

### The `split` Command — Every Flag Dissected

```bash
split \
    --lines=100 \           # how many lines per chunk
    --suffix-length=2 \     # how long the suffix is (aa, ab...)
    --additional-suffix="" \ # optional file extension (.log, .txt)
    /var/log/app/access.log \ # INPUT file (never modified)
    /tmp/log_parts/access_part_  # OUTPUT prefix (dir + filename prefix)
```

Short form — same thing:
```bash
split -l 100 -a 2 /var/log/app/access.log /tmp/log_parts/access_part_
```

```
Input  : /var/log/app/access.log         ← read-only, never touched
Output : /tmp/log_parts/access_part_aa   ← new files created here
                         access_part_ab
                         access_part_ac
                         access_part_ad
```

---

### Three Split Modes — Lines vs Bytes vs Chunks

```bash
# BY LINES (this task) — keeps log entries whole
split -l 100 file.log prefix_
# Each file = exactly 100 complete lines
# Last file = remainder (75 in our case)
# ✓ Best for log analysis — entries never cut mid-line

# BY BYTES — fixed size chunks
split -b 500M file.log prefix_
# Each file = exactly 500MB
# ✗ Can cut a log entry in half mid-line — bad for parsing

# BY NUMBER OF CHUNKS — equal-ish pieces
split -n 4 file.log prefix_
# Splits into exactly 4 parts (may cut mid-line)
# ✗ Also bad for log analysis
```

**Rule:** For log files → always use `-l` (lines). Never `-b` (bytes).

---

### The Suffix System — How `aa` → `ab` → `zz` Works

```
Default suffix length = 2  →  26 × 26 = 676 max files
                                         (handles 67,600 lines at 100/chunk)

Sequence:
aa, ab, ac, ad, ae, af, ag, ah, ai, aj, ak, al, am,
an, ao, ap, aq, ar, as, at, au, av, aw, ax, ay, az,
ba, bb, bc, bd ... bz,
ca ... zz

If you need more chunks:
split -l 100 -a 3 file.log prefix_   # aaa → zzz = 17,576 files
split -l 100 -a 4 file.log prefix_   # aaaa → zzzz = 456,976 files
```

Adding a file extension:
```bash
split -l 100 --additional-suffix=".log" access.log access_part_
# Creates: access_part_aa.log, access_part_ab.log ...
```

---

### Why MD5 Verification Matters

```
BEFORE split:  md5sum /var/log/app/access.log
               → 7cf634a804f5a660f31aba1c9eb7b262

AFTER split, reconstruct:
               cat /tmp/log_parts/access_part_* > /tmp/rebuilt.log
               md5sum /tmp/rebuilt.log
               → 7cf634a804f5a660f31aba1c9eb7b262  ← identical ✓

Why this proves integrity:
MD5 = a 128-bit hash of every byte in the file
Even 1 changed character = completely different hash
Matching MD5 = byte-for-byte identical content
```

In incident response, this matters legally — you prove the evidence wasn't tampered with.

---

### Other Tools for Big File Analysis (Without Splitting)

```bash
# Process without loading into memory
grep "ERROR" /var/log/app/access.log   # streams line by line ✓
awk '/500/{print $1}' access.log       # also streams ✓
sed -n '1000,2000p' access.log         # print lines 1000-2000 ✓

# Read beginning / end without opening whole file
head -100 access.log                   # first 100 lines
tail -100 access.log                   # last 100 lines
tail -f access.log                     # live follow

# Compressed analysis (never decompress first)
zcat access.log.gz | grep "ERROR"      # read .gz directly
zgrep "ERROR" access.log.gz            # grep inside .gz
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "What does the `split` command do and what are its main options?"

**Answer:**
> *"`split` divides a file into smaller pieces without modifying the original. The three main splitting modes are: `-l` for lines (best for log files — keeps entries intact), `-b` for bytes (fixed file sizes), and `-n` for a fixed number of chunks. The output prefix determines where files are saved and what they're named — `split` appends the suffix automatically. For log analysis I always use `-l` because cutting by bytes could split a log entry across two files, breaking parsers."*

```bash
split -l 100 access.log /tmp/parts/access_part_   # 100 lines each
split -b 500M bigfile.dat /tmp/parts/chunk_       # 500MB each
split -n 4   access.log /tmp/parts/part_          # exactly 4 parts
```

---

### Q2. "Why use `split` instead of just using `grep` or `awk` on the full file?"

**Answer:**
> *"For a single query, `grep` and `awk` stream the file fine — they don't load it all into RAM. But during an incident investigation you often need to run many different analyses, share parts with teammates in different timezones, or feed files into tools that have memory limits like certain log parsers or Excel. Splitting enables true parallel processing — four analysts can simultaneously work on different time windows. It also lets you isolate the time range of an incident — if the spike happened between 9-10AM, you only need `access_part_ab` rather than processing the entire 2.5GB file every time."*

---

### Q3. "How does `split` name files? How would you get 1000+ parts?"

**Answer:**
> *"By default `split` uses a 2-character alphabetical suffix — `aa` through `zz` — giving 676 possible files. At 100 lines each that handles up to 67,600 lines. For larger files I increase the suffix length with `-a`:"*

```bash
split -l 100 -a 3 access.log prefix_   # aaa→zzz = 17,576 files
split -l 100 -a 4 access.log prefix_   # aaaa→zzzz = 456,976 files
```

> *"You can also use numeric suffixes with `-d` flag — gives `00`, `01`, `02` which is easier to sort numerically and more human-readable for scripting:"*

```bash
split -l 100 -d -a 4 access.log access_part_
# Creates: access_part_0000, access_part_0001, ...
```

---

### Q4. "How do you verify the split files contain all the data and nothing is lost?"

**Answer:**
> *"Two checks: line count and MD5 checksum. First confirm the sum of all parts equals the original line count. Then reconstruct using `cat` and compare MD5 hashes — if they match byte-for-byte, the data is complete and intact."*

```bash
# Check 1 — line counts
wc -l /var/log/app/access.log         # original: 375
wc -l /tmp/log_parts/access_part_*   # parts: 100+100+100+75 = 375 ✓

# Check 2 — MD5 integrity
ORIG=$(md5sum /var/log/app/access.log | awk '{print $1}')
cat /tmp/log_parts/access_part_* | md5sum   # must match $ORIG ✓
```

---

### Q5. "What other commands can process large log files without splitting them?"

**Answer:**
> *"`grep`, `awk`, and `sed` all stream files line by line — they never load the whole file into memory, so they work on files of any size. For compressed logs, `zcat` and `zgrep` process gzip files directly without decompressing to disk. For really large files I'd also use `parallel` from GNU parallel to run grep on all split parts simultaneously, or `ripgrep` which is significantly faster than grep on large files due to SIMD optimizations."*

```bash
grep "500" access.log                     # stream — any size ✓
awk '$9 == 500 {print $1}' access.log     # stream — any size ✓
zgrep "ERROR" access.log.gz               # compressed — no temp file ✓

# Parallel grep across all parts:
ls /tmp/log_parts/ | parallel grep "500" /tmp/log_parts/{}
```

---

### Q6. "How would you split a log file by date/time range instead of by line count?"

**Answer:**
> *"For time-based splitting, `awk` is the right tool — `split` only knows about lines and bytes, not content. I'd match the timestamp pattern in each log line and redirect to different files:"*

```bash
# Split access.log by hour
awk '{
    match($4, /\[([^:]+):([0-9]+):/, arr)
    hour = arr[2]
    print > "/tmp/log_parts/access_hour_" hour ".log"
}' /var/log/app/access.log

# Result: access_hour_08.log, access_hour_09.log, access_hour_10.log ...
```

> *"This is actually more useful for incident investigation than fixed line counts — if an incident happened at 09:15, I want `access_hour_09.log` regardless of how many lines it has."*

---

### Q7. "In a real incident at Amazon scale, how would you handle a 500GB log file?"

**Answer — show scale thinking:**

> *"At 500GB, even `split` producing 5 million files becomes unwieldy. The approach changes:"*

```bash
# Step 1 — narrow by time range first (most incidents = short window)
grep "03/May/2026:09:" access.log > /tmp/incident_window.log
# Now working with maybe 50MB instead of 500GB

# Step 2 — stream directly to analysis without storing
grep "500\|503\|504" access.log | awk '{print $1}' | sort | uniq -c | sort -rn
# Find error-generating IPs without ever writing temp files

# Step 3 — use parallel processing for scanning
split -l 1000000 -d access.log /tmp/chunk_
ls /tmp/chunk_* | xargs -P8 -I{} grep "ERROR" {} >> /tmp/all_errors.log
# -P8 = 8 parallel grep processes
```

> *"In AWS specifically, for truly massive logs I'd push to S3 and use Athena for SQL queries over the raw log files — no local storage needed, distributed query across the entire dataset. For CloudWatch Logs, `aws logs filter-log-events` with time range and filter pattern handles it without downloading anything."*

---

## Part 5: Cheat Sheet

```
CORE COMMAND:
  split -l 100 input.log /output/dir/prefix_
  split -l 100 -a 2 input.log prefix_      # explicit suffix length
  split -l 100 -d    input.log prefix_      # numeric suffix (00,01...)

KEY FLAGS:
  -l N    → N lines per file        (best for logs)
  -b SIZE → SIZE bytes per file     (500M, 1G, 100K)
  -n N    → exactly N output files
  -a N    → suffix length (default 2 = aa..zz = 676 files)
  -d      → numeric suffix instead of alpha
  --additional-suffix=.log  → add extension to output files

SUFFIX CAPACITY:
  -a 2  →   676 files  (handles ~67K lines at 100/chunk)
  -a 3  → 17,576 files (handles ~1.7M lines)
  -a 4  → 456,976 files

VERIFY INTEGRITY:
  wc -l parts/*                      # sum must equal original
  cat parts/* | md5sum               # must match original md5sum
  diff original <(cat parts/*)       # byte-level diff

RECONSTRUCT:
  cat /tmp/log_parts/access_part_* > /tmp/rebuilt.log

LARGE FILE TOOLS:
  grep/awk/sed    → stream any size, no RAM issues
  zcat/zgrep      → process .gz without decompressing
  head/tail -n N  → read N lines from start/end
  sed -n '100,200p' file  → extract specific line range
  xargs -P8       → parallel processing across parts
```

> **Amazon interview tip:** They'll push you to think at scale. Mention that for multi-GB files the bottleneck shifts from tools to strategy — narrow the time window first with `grep` before any heavy processing, and bring up S3 + Athena for truly large-scale log analysis. Showing you think beyond the single-server solution is what they're looking for at Amazon's scale.
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Log File Volume Assessment
> **Company:** JPMorgan | **Difficulty:** Easy
---
#### **Scenario**
The `/var` directory contains logs from multiple applications, and cleanup planning is needed. Some applications create their own subdirectories with nested log files.
#### **Task**
Find and count all files ending with the `.log` extension anywhere under `/var` including subdirectories, save the total count to `/home/devops/log_count.txt`, and use standard Linux commands to output only the total number of `.log` files found. Additionally, identify `.log` files **larger than 512 KB** and save the **count** to `/home/devops/large_log_count.txt`
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/log-file-volume-assessment)

## Log File Volume Assessment — Full Deep Dive

---

## Part 1: Understand It Simply

### What Are We Doing and Why?

```
/var directory tree:
  /var/log/
  ├── syslog.log           ← system logs
  ├── auth.log             ← auth events (not .log extension — skip)
  ├── app/
  │   ├── access.log       ← app access log
  │   └── error.log        ← app error log
  ├── nginx/
  │   ├── access.log
  │   └── error.log
  └── mysql/
      └── mysql-slow.log

JPMorgan cleanup scenario:
  "How many .log files exist?"      → total count → log_count.txt
  "Which ones are bloated (>512KB)?"→ large count → large_log_count.txt
```

Two commands power this entire task:

```
find   →  walks directory trees, filters by name/size/type
wc -l  →  counts lines (one result per line = one file per line)
```

---

## Part 2: Build and Execute

### Step 1 — Simulate the Environment### Step 2 — Show the Problem Clearly First### Step 3 — Core Task: Count ALL `.log` Files### Step 4 — Save Total Count to File### Step 5 — Find Files Larger Than 512KB### Step 6 — Save Large Count and Final VerificationBoth files saved correctly. Now let's understand everything deeply.

---

## Part 3: Every Concept Explained Simply

### How `find` Works — The Mental Model

```
find  /var         -type f       -name "*.log"
  ↑                   ↑               ↑
WHERE to look    WHAT kind      WHAT pattern

find walks the ENTIRE tree recursively:
  /var
  /var/log             ← checks: is it a file? name ends .log?
  /var/log/access.log  ← ✓ file ✓ .log  → output this
  /var/log/btmp        ← ✓ file ✗ .log  → skip
  /var/log/app/        ← ✗ directory    → skip (but descend into it)
  /var/log/app/err.log ← ✓ file ✓ .log  → output this
```

### Breaking Down Every Flag Used

```bash
find /var -type f -name "*.log" | wc -l

find    →  the command
/var    →  starting directory (recursive from here)
-type f →  f = regular file only
            d = directory
            l = symlink
-name "*.log"  →  glob pattern matching filename only
                   * = any characters
                   must end exactly in .log
                   (syslog.log.gz does NOT match → excluded ✓)
| wc -l →  pipe: each found file = 1 line → count lines = count files
```

```bash
find /var -type f -name "*.log" -size +512k

-size +512k  →  + = strictly greater than
                512 = the value
                k = unit (kilobytes)

Units:
  c  →  bytes
  k  →  kilobytes (1024 bytes)
  M  →  megabytes (1024 KB)
  G  →  gigabytes

+512k  = files LARGER than 512KB
-512k  = files SMALLER than 512KB
 512k  = files EXACTLY 512KB (rarely used)
```

---

### Why `wc -l` Counts Files

```
find /var -type f -name "*.log" outputs:
  /var/log/access.log        ← line 1
  /var/log/app/error.log     ← line 2
  /var/log/nginx/access.log  ← line 3
  ...

One file = one line
wc -l counts lines = counts files

wc -l   →  count lines
wc -w   →  count words
wc -c   →  count characters/bytes
wc -m   →  count characters (multibyte aware)
```

---

### The `*.log` Pattern — What It Matches and Doesn't

```
MATCHES (counted ✓):
  access.log          ← ends in .log
  mysql-slow.log      ← ends in .log
  app.2026-05-03.log  ← ends in .log
  .log                ← starts with nothing, ends in .log

DOES NOT MATCH (excluded ✓):
  syslog.log.gz       ← ends in .gz not .log
  access.log.1        ← ends in .1 not .log
  logfile             ← no extension
  .log.xz             ← ends in .xz

The * in *.log means "zero or more of any character"
The .log must be at the END — exact suffix match
```

---

### `find -exec ls -lh` — Inspect File Details Inline

```bash
find /var -type f -name "*.log" -size +512k \
     -exec ls -lh {} \;

-exec         →  run a command for each found file
ls -lh        →  list with human-readable sizes
{}            →  placeholder for the current file path
\;            →  end of -exec command (escaped semicolon)

Output:
  -rw-r--r-- 1 root root 1.5M ... /var/log/mysql/app.log
  -rw-r--r-- 1 root root 2.0M ... /var/log/java-app/app.log
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "What does `find -type f -name "*.log"` do? Why `-type f`?"

**Answer:**
> *"`find` recursively walks a directory tree and outputs paths matching your criteria. `-name "*.log"` filters by filename pattern — only files ending exactly in `.log`. `-type f` restricts to regular files only, excluding directories, symlinks, and device files. Without `-type f`, if someone created a directory called `app.log/`, it would appear in results and corrupt the count. The `-type f` is defensive programming — always include it in production scripts."*

---

### Q2. "Explain `| wc -l`. Why does it count files?"

**Answer:**
> *"`find` outputs one file path per line — each match is on its own newline. Piping to `wc -l` counts those newlines, which equals the number of files found. It's a simple and reliable pattern. The only edge case is filenames with embedded newlines — extremely rare in practice, but for bulletproof scripts you'd use `find -print0 | xargs -0 wc -l` with null-delimited output instead."*

```bash
# Standard (works 99.9% of the time)
find /var -type f -name "*.log" | wc -l

# Bulletproof (handles filenames with newlines)
find /var -type f -name "*.log" -print0 | tr -cd '\0' | wc -c
```

---

### Q3. "What does `-size +512k` mean? What are the other size units?"

**Answer:**
> *"The `-size` flag filters files by size. The prefix controls comparison: `+` means strictly greater than, `-` means less than, and no prefix means exactly equal. The suffix is the unit — `c` for bytes, `k` for kilobytes (1024 bytes), `M` for megabytes, `G` for gigabytes. One important nuance: `find` rounds up to the nearest block — so `-size +512k` means 'greater than 512 blocks of 1KB' which in practice means files whose size rounds up above 512KB."*

```bash
find /var -name "*.log" -size +512k    # larger than 512KB
find /var -name "*.log" -size +100M    # larger than 100MB
find /var -name "*.log" -size -10k     # smaller than 10KB
find /var -name "*.log" -size +1G      # larger than 1GB
```

---

### Q4. "What's the difference between `-name` and `-iname` in find?"

**Answer:**
> *"`-name` is case-sensitive — `*.log` won't match `Error.LOG` or `access.Log`. `-iname` is case-insensitive — it matches regardless of case. In a JPMorgan environment where multiple teams may have deployed apps with inconsistent naming conventions, `-iname` is safer for log discovery:"*

```bash
find /var -type f -name "*.log"   # matches: error.log, NOT Error.LOG
find /var -type f -iname "*.log"  # matches: error.log AND Error.LOG AND ERROR.LOG
```

---

### Q5. "How would you find the largest `.log` files specifically?"

**Answer:**
> *"Combine `find` with `sort` by size. Two approaches — using `ls -s` for quick sorts or `du` for accurate disk usage:"*

```bash
# Method 1 — find + ls + sort (human-readable)
find /var -type f -name "*.log" -exec ls -lh {} \; \
  | sort -k5 -rh | head -10
#                    ↑ sort by column 5 (size), reverse, human-readable

# Method 2 — find + du + sort (most accurate)
find /var -type f -name "*.log" \
  | xargs du -sh 2>/dev/null \
  | sort -rh | head -10

# Method 3 — find with printf (cleanest output)
find /var -type f -name "*.log" -printf "%s %p\n" \
  | sort -rn | head -10 \
  | awk '{printf "%.1fMB  %s\n", $1/1048576, $2}'
```

---

### Q6. "How would you find `.log` files NOT modified in the last 30 days (cleanup candidates)?"

**Answer:**
> *"Use `-mtime` — modification time in 24-hour blocks. `+30` means more than 30 days ago. Combine with size filter for files worth cleaning:"*

```bash
# Files older than 30 days
find /var -type f -name "*.log" -mtime +30

# Old AND large — best cleanup targets
find /var -type f -name "*.log" -mtime +30 -size +100M

# Show size + age together
find /var -type f -name "*.log" -mtime +30 \
  -printf "%-40p  %kKB  modified %Td days ago\n" \
  | sort -t' ' -k2 -rn | head -20

# Safe delete — preview first, then delete
find /var -type f -name "*.log" -mtime +30 -size +100M   # preview
find /var -type f -name "*.log" -mtime +30 -size +100M -delete  # execute
```

---

### Q7. "What's the difference between `-mtime`, `-atime`, and `-ctime`?"

**Answer:**

| Flag | Tracks | Updated when |
|------|--------|--------------|
| `-mtime` | **m**odification time | file content changes |
| `-atime` | **a**ccess time | file is read |
| `-ctime` | **c**hange time | permissions/ownership/content changes |

> *"For log cleanup, `-mtime` is what you want — it tracks when the log was last written to. `-atime` is unreliable on many systems because `noatime` mount option disables it for performance. `-ctime` isn't quite 'creation time' — it's metadata change time, which confuses most people."*

```bash
find /var -name "*.log" -mtime +30    # not written to in 30 days
find /var -name "*.log" -mtime -1     # modified in last 24 hours (active logs)
find /var -name "*.log" -mtime +7 -mtime -30  # between 7 and 30 days old
```

---

### Q8. "How would you produce a full log inventory report — not just a count?"

**Answer:**
> *"Combine `find` with `printf` for structured output, or pipe through `awk` and `sort` for a formatted report:"*

```bash
echo "Log Inventory Report — $(date)"
echo "======================================"
printf "%-60s %8s %12s\n" "FILE" "SIZE" "LAST MODIFIED"
echo "----------------------------------------------------------------------"

find /var -type f -name "*.log" \
  -printf "%-60p %8k KB %Td/%Tm/%TY\n" \
  | sort -k2 -rn

echo ""
echo "SUMMARY"
echo "-------"
echo "Total .log files : $(find /var -type f -name "*.log" | wc -l)"
echo "Over 512KB       : $(find /var -type f -name "*.log" -size +512k | wc -l)"
echo "Over 100MB       : $(find /var -type f -name "*.log" -size +100M | wc -l)"
echo "Older than 30d   : $(find /var -type f -name "*.log" -mtime +30 | wc -l)"
```

---

## Part 5: Cheat Sheet

```
CORE COMMANDS:
  find /var -type f -name "*.log"            → list all .log files
  find /var -type f -name "*.log" | wc -l    → count them
  find /var -type f -name "*.log" -size +512k → larger than 512KB

SAVE TO FILE (number only):
  find /var -type f -name "*.log" | wc -l > /home/devops/log_count.txt
  find /var -type f -name "*.log" -size +512k | wc -l > large_log_count.txt

SIZE UNITS IN find:
  c  = bytes
  k  = kilobytes (1024 bytes)
  M  = megabytes
  G  = gigabytes
  +  prefix = greater than
  -  prefix = less than
  (none) = exactly equal

TIME FLAGS:
  -mtime +30  = not modified in 30+ days
  -mtime -1   = modified in last 24 hours
  -newer file = newer than a reference file

NAME vs INAME:
  -name  = case-sensitive  (*.log ≠ *.LOG)
  -iname = case-insensitive (*.log = *.LOG)

SORT BY SIZE:
  find /var -name "*.log" | xargs du -sh | sort -rh | head -10

PRINT WITH METADATA:
  find /var -name "*.log" -printf "%s %p\n"   → bytes + path
  find /var -name "*.log" -printf "%k %p\n"   → KB + path
  find /var -name "*.log" -printf "%TD %p\n"  → date + path
```

> **JPMorgan interview tip:** Finance firms care about audit trails and compliance — they'll appreciate if you mention saving findings to a file before any cleanup action, and suggest scheduling this as a cron job for ongoing monitoring (`0 6 * * * find /var -type f -name "*.log" | wc -l >> /var/log/log_inventory.csv`). Proactive monitoring over reactive firefighting is the mindset they want to see.
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Manage Service Failure Recovery
Company: Apple | Difficulty: Hard
Scenario
You have a shell script at /usr/local/bin/check_[app.sh](http://app.sh) that runs periodically and exits with a non-zero code. The script is currently failing due to a simulated error condition.
Task
Create a systemd service named check_app.service that automatically restarts the script when it fails, but stops retrying after 3 restart attempts within 60 seconds. Configure the service to start on boot with a 5-second delay between restart attempts, then start the service and verify it hits the restart limit.

## Manage Service Failure Recovery — Full Deep Dive

---

## Part 1: Understand It Simply (The Story)

### What Problem Are We Solving?

```
Without restart logic:
  check_app.sh crashes at 03:00 AM
          ↓
  Service stays dead until someone manually restarts it
          ↓
  App is down for hours — users affected

With systemd restart logic:
  check_app.sh crashes at 03:00 AM
          ↓
  systemd detects exit code ≠ 0
          ↓
  Waits 5 seconds → restarts automatically
  Crash again → wait 5s → restart
  Crash again → wait 5s → restart
          ↓
  After 3 attempts in 60 seconds → GIVES UP
  (prevents infinite crash loop hammering the system)
          ↓
  Sends alert → on-call engineer wakes up
```

### The Three systemd Restart Controls

```
RestartSec=5          →  wait 5 seconds between each restart attempt
                          prevents rapid crash loops

StartLimitBurst=3     →  allow max 3 start attempts...
StartLimitIntervalSec=60 →  ...within any 60-second window

When limit hit: service enters "failed" state, stops retrying
```

### Visual Timeline

```
t=0s    Service starts → crashes (exit code 1)
t=5s    Restart #1    → crashes (exit code 1)
t=10s   Restart #2    → crashes (exit code 1)
t=15s   Restart #3    → crashes (exit code 1)
        ↑
        3 crashes in 15 seconds < 60 second window
        → LIMIT HIT → systemd stops → "failed" state
        No more automatic restarts
```

---

## Part 2: Build and Execute

### Step 1 — Create the Failing Script### Step 2 — Create the systemd Service Unit File### Step 3 — Reload, Enable, and Validate### Step 4 — Simulate systemd Behavior (Container-Safe)### Step 5 — Show What systemd Commands Look Like in ProductionEverything working exactly as designed. Now let's understand every piece deeply.

---

## Part 3: Every Concept Explained Simply

### The Full systemd Unit File Structure

```ini
[Unit]          ← METADATA: what is this service, dependencies, limits
[Service]       ← BEHAVIOUR: how to run it, restart policy, environment
[Install]       ← BOOT: when to start it during system startup
```

Each section answers a different question:

```
[Unit]    →  "What is this and when should it run?"
[Service] →  "How exactly should systemd manage this process?"
[Install] →  "Should this start on boot, and for which target?"
```

---

### `Restart=` Options — Full Comparison

```
Restart=no           → never restart (default)
                       Use for: one-shot tasks, scripts that run once

Restart=always       → restart on ANY exit — clean or failure
                       Use for: daemons that must ALWAYS be running

Restart=on-failure   → restart ONLY when exit code ≠ 0 or crash signal
                       Use for: services that sometimes exit cleanly
                                (our case — script should restart when broken)

Restart=on-success   → restart ONLY when exit code = 0
                       Use for: polling loops that restart after success

Restart=on-abnormal  → restart on crash signal, timeout, watchdog — NOT clean exit
                       Use for: strict services where manual stop = intentional

Restart=on-abort     → only on unhandled signals (SIGABRT, SIGSEGV)
                       Use for: C applications that shouldn't normally crash
```

---

### `StartLimitBurst` and `StartLimitIntervalSec` — The Rate Limiter

```
StartLimitBurst=3          ←  maximum 3 start attempts
StartLimitIntervalSec=60s  ←  within any 60-second sliding window

     Timeline:
     ┌──────────────── 60 second window ─────────────────┐
     start #1   start #2   start #3
     t=0s       t=5s       t=10s
       ↓          ↓          ↓
     CRASH      CRASH      CRASH
                            ↑
                     3 crashes in 10s < 60s
                     → LIMIT HIT → FAILED state
     └────────────────────────────────────────────────────┘

What if crashes are spread out?
     t=0s    CRASH  (attempt 1)
     t=5s    CRASH  (attempt 2)
     ...
     t=65s   CRASH  (attempt 3 — but window reset at 60s!)
     → Only 2 in last 60s → still under limit → keeps trying
```

---

### `Type=` — How systemd Tracks the Process

```
Type=simple    → PID from ExecStart IS the service
                 systemd watches it directly
                 Most common for scripts and simple daemons

Type=forking   → ExecStart forks a child and exits
                 systemd tracks the child (needs PIDFile=)
                 Old-style daemons (Apache 2.2, older MySQL)

Type=notify    → Process signals systemd when READY
                 sd_notify() call from the application
                 More reliable "ready" detection

Type=oneshot   → Runs once and exits (like a script)
                 systemd waits for it to finish
                 RemainAfterExit=yes to show "active" after

Type=exec      → Like simple but waits for exec() to succeed
                 Better for wrapper scripts

For our script: Type=simple is correct ✓
  The script IS the service — systemd watches its PID directly
```

---

### `WantedBy=multi-user.target` — What Does Enable Actually Do?

```
systemctl enable check_app.service

What it does physically:
  Creates a SYMLINK:
  /etc/systemd/system/multi-user.target.wants/
    check_app.service → /etc/systemd/system/check_app.service

Boot sequence:
  systemd starts → reaches multi-user.target
  → reads .wants/ directory
  → finds check_app.service symlink
  → starts it automatically

systemctl disable = removes the symlink
systemctl start  = starts RIGHT NOW (regardless of enable)
systemctl enable = starts ON NEXT BOOT

enable + start = both immediately AND on every future boot ✓
```

---

### `systemctl reset-failed` — Why You Need It

```
After hitting StartLimitBurst:
  Service state = "failed" (start-limit-hit)
  systemd REFUSES to start it again automatically

This is intentional — prevents infinite crash hammering

To restart after fixing the underlying problem:
  1. Fix the script (remove the bug)
  2. sudo systemctl reset-failed check_app.service
     → clears the failed state, resets the counter
  3. sudo systemctl start check_app.service
     → starts fresh with full 3 retries available again

Skipping reset-failed → systemctl start still works
BUT the counter isn't reset → may hit limit faster
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "Explain the three restart-related directives in this service file."

**Answer:**
> *"Three separate controls work together. `Restart=on-failure` tells systemd to attempt restart when the process exits with non-zero code or crashes — not on a clean exit. `RestartSec=5s` introduces a 5-second cooling-off delay between each attempt, preventing a tight crash loop that would hammer the system or fill logs. `StartLimitBurst=3` combined with `StartLimitIntervalSec=60s` is the circuit breaker — after 3 start attempts within any 60-second window, systemd stops retrying entirely and marks the service as `failed`. This is the critical safety mechanism that prevents runaway restart loops at 3 AM."*

---

### Q2. "What's the difference between `systemctl enable` and `systemctl start`?"

**Answer:**
> *"`start` starts the service immediately right now, once. `enable` creates a symlink in the appropriate `.wants/` directory so the service starts automatically on every boot — it doesn't start the service right now. For production deployments you almost always want both: `enable` to survive reboots, `start` to begin immediately without waiting for the next boot. `disable` removes the symlink (stops auto-start on boot) but doesn't stop a currently running service — for that you'd use `stop`."*

```bash
systemctl start check_app    # run NOW only
systemctl enable check_app   # run on BOOT only
systemctl enable --now check_app  # run NOW + on every boot ✓
```

---

### Q3. "What does `Type=simple` mean and when would you use `Type=forking`?"

**Answer:**
> *"`Type=simple` means the process launched by `ExecStart` is the main service process — systemd watches that PID directly. When it exits, systemd knows the service ended. `Type=forking` is for old-style Unix daemons that fork a child process and then the parent exits — the service is actually the child. systemd needs a `PIDFile=` to track which PID to watch. In modern infrastructure, `Type=simple` is almost always correct for new services. `Type=notify` is even better for complex services that need to signal systemd when they're truly ready to serve traffic — nginx and PostgreSQL use this."*

---

### Q4. "What happens when `StartLimitBurst` is hit? How do you recover?"

**Answer:**
> *"When the limit is hit, systemd marks the service as `failed` with result `start-limit-hit` and stops all automatic restart attempts. The service stays dead even if `Restart=always` is configured — the rate limiter overrides it. To recover you need to: fix the underlying problem in the script or application, then run `systemctl reset-failed check_app.service` to clear the failed state and reset the counter, then `systemctl start check_app.service` to bring it back up. If you skip `reset-failed`, the counter carries over and you might hit the limit faster on the next failure."*

```bash
sudo systemctl reset-failed check_app.service  # clear failed state
sudo systemctl start check_app.service         # restart fresh
sudo journalctl -fu check_app.service          # watch in real time
```

---

### Q5. "What's `After=network.target` doing in `[Unit]`?"

**Answer:**
> *"`After=` defines ordering — it tells systemd to wait until `network.target` is reached before starting this service. It doesn't mean the service requires the network — just that if both are being started, the network comes first. For a health check script that connects to a database or makes API calls, starting before the network is ready would cause immediate failures. Common ordering dependencies are `network.target` (basic network up), `network-online.target` (full network connectivity), and `postgresql.service` or `mysql.service` for apps that need a specific database."*

---

### Q6. "How would you watch the service fail and restart in real time?"

**Answer:**
> *"`journalctl -fu check_app.service` is the go-to command — `-f` follows (like `tail -f`), `-u` filters to the specific unit. You'll see each crash, the restart delay, and eventually the `start-limit-hit` message. For more detail, `systemctl status check_app.service` gives a snapshot with the last few log lines and the current state. During an incident I'd have both open in split terminals — journalctl in one for the live stream, systemctl status in the other for the state machine view."*

```bash
journalctl -fu check_app.service           # live follow
journalctl -u check_app.service --since "10 min ago"  # recent history
systemctl status check_app.service         # current state snapshot
```

---

### Q7. "How would you add an alert when the service hits the failure limit?"

**Answer:**
> *"Use `OnFailure=` in the `[Unit]` section to trigger another service when this one fails. The cleanest approach is a dedicated notification service:"*

```ini
# In check_app.service [Unit] section:
OnFailure=notify-failure@%n.service

# Create /etc/systemd/system/notify-failure@.service
[Unit]
Description=Failure notification for %i

[Service]
Type=oneshot
ExecStart=/usr/local/bin/notify.sh %i
```

```bash
# /usr/local/bin/notify.sh
#!/bin/bash
SERVICE=$1
echo "ALERT: $SERVICE hit restart limit on $(hostname) at $(date)" \
  | mail -s "Service Failed: $SERVICE" ops@apple.com

# Or post to Slack/PagerDuty webhook
curl -X POST https://hooks.slack.com/... \
  -d "{\"text\": \"$SERVICE failed on $(hostname)\"}"
```

---

### Q8. "What's the difference between `Restart=on-failure` and `Restart=always`?"

**Answer:**
> *"`Restart=on-failure` only restarts when the exit code is non-zero or there's a crash signal — a clean `exit 0` stops the restarts. `Restart=always` restarts regardless of exit code — even a clean exit triggers a restart. For our health check script that always fails, both would behave identically. But for a service that sometimes exits cleanly as intended — like a migration script that should run once and stop — `always` would cause an infinite loop of successful runs. For most real services, `on-failure` is the safer default."*

---

## Part 5: Cheat Sheet

```
UNIT FILE STRUCTURE:
  [Unit]    → description, ordering, rate limits
  [Service] → ExecStart, Restart, Type, Environment
  [Install] → WantedBy (boot target)

KEY DIRECTIVES:
  ExecStart=        → command to run
  Type=simple       → PID from ExecStart is the service
  Restart=on-failure → restart on non-zero exit / crash
  RestartSec=5s     → wait 5s between restart attempts
  StartLimitBurst=3 → max 3 attempts...
  StartLimitIntervalSec=60s → ...in any 60s window
  After=network.target → start after networking
  WantedBy=multi-user.target → enable for normal boot

RESTART= OPTIONS:
  no          → never restart
  always      → always restart (even on clean exit)
  on-failure  → restart on non-zero exit or crash ← most common
  on-abnormal → only on crash signals/timeout

ESSENTIAL COMMANDS:
  systemctl daemon-reload          → reload unit files after edits
  systemd-analyze verify <file>    → validate syntax
  systemctl enable check_app       → start on boot (adds symlink)
  systemctl start check_app        → start now
  systemctl enable --now check_app → both at once ← best practice
  systemctl status check_app       → current state
  journalctl -fu check_app         → live log stream
  systemctl reset-failed check_app → clear failed state to retry

STATES TO KNOW:
  active (running)      → healthy, running normally
  failed                → crashed and exceeded restart limit
  failed (start-limit-hit) → rate limiter triggered ← our case
  activating (auto-restart) → between restart attempts

RECOVERY WORKFLOW:
  1. Fix the underlying issue
  2. systemctl reset-failed check_app
  3. systemctl start check_app
  4. journalctl -fu check_app  (watch it recover)
```

> **Apple interview tip:** They care deeply about **reliability engineering** — mention `OnFailure=` for alerting, `WatchdogSec=` for liveness monitoring, and the distinction between `enable` (persistent) vs `start` (immediate). Showing you think about what happens *after* the limit is hit — the alert, the reset workflow, the root cause investigation — separates a senior SRE answer from a junior "I added Restart=on-failure" answer.

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Managing Process Overload
> **Company:** [Booking.com](http://Booking.com) | **Difficulty:** Medium
---
#### Scenario:
The process count on a production server keeps increasing even though no new workloads were deployed. Zombie processes are accumulating in the system.
#### Task:
Find all zombie processes, kill their parent processes to remove the zombies, and confirm they are gone from the system.
####  Expected Output:
No leftover or defunct processes should be left after the fix.

## Managing Process Overload — Full Deep Dive

---

## Part 1: Understand It Simply (The Story)

### What is a Zombie Process?

```
Normal process lifecycle:
  Parent spawns Child
       ↓
  Child does work
       ↓
  Child finishes → calls exit()
       ↓
  Parent calls wait() → reads child's exit code
       ↓
  Child entry REMOVED from process table ✓

Zombie process lifecycle:
  Parent spawns Child
       ↓
  Child does work
       ↓
  Child finishes → calls exit()
       ↓
  Parent NEVER calls wait()  ← THE BUG
       ↓
  Child stays in process table as <defunct>
  Using NO CPU, NO memory — just a PID slot
       ↓
  Thousands accumulate → PID table exhausted
  → "Cannot fork" → new processes can't start
```

### Why Can't You Kill a Zombie Directly?

```
kill -9 <zombie_pid>   ←  DOES NOTHING

Reason: the zombie is already DEAD
        It's just a record waiting to be collected
        Like a death certificate waiting to be filed

The only fix:
  Kill or fix the PARENT → parent's cleanup code runs
  → parent finally calls wait() → zombie disappears
  OR
  Parent dies → init/systemd (PID 1) adopts zombie
  → init calls wait() → zombie cleaned up automatically
```

### Visual: Process Table

```
  PID   PPID  STAT  COMMAND
    1      0   Ss   systemd          ← init, PID 1
  501      1   S    bash             ← parent (the bug is here)
  502    501   Z    [worker] <defunct> ← zombie #1 — child of 501
  503    501   Z    [worker] <defunct> ← zombie #2 — child of 501
  504    501   Z    [worker] <defunct> ← zombie #3 — child of 501

Fix: kill 501 (the parent bash)
  → zombies 502, 503, 504 get adopted by systemd
  → systemd calls wait() → all three disappear ✓
```

---

## Part 2: Build and Execute

### Step 1 — Create Zombie Processes to Simulate the ProblemZombies created. Now the full investigation and cleanup.

### Step 2 — Investigate: Find All Zombies5 zombies confirmed. Now the full investigation workflow.

### Step 3 — Full Investigation: Map Zombies to Their Parents5 zombies with parent 837. Now the complete fix workflow.

### Step 4 — Full Investigation Report and FixZero zombies. System clean. Now the deep explanation.

---

## Part 3: Every Concept Explained Simply

### The Complete Process Lifecycle

```
HEALTHY lifecycle:
  ┌─────────┐   fork()   ┌─────────┐
  │ PARENT  │ ─────────► │  CHILD  │  ← new PID, new process
  └─────────┘            └─────────┘
       │                      │
       │                      │ does work...
       │                      │ calls exit(0)
       │                      ▼
       │               [ ZOMBIE STATE ] ← stays here waiting
       │                 exit code = 1
       │                 "Please collect me"
       │
       ├─ waitpid() ─────────────────────────────┐
       │              Parent reads exit code      │
       │              Process table entry REMOVED ┘
       │
       ▼
  Parent continues
```

### What Lives in a Zombie

```
A zombie process holds:
  ✓ PID  (occupies a slot in the PID table)
  ✓ Exit code  (waiting for parent to read it)
  ✓ Basic accounting info (CPU time used, etc.)

A zombie process has ZERO:
  ✗ Memory pages
  ✗ CPU time
  ✗ File descriptors
  ✗ Executable code running

This is why kill -9 doesn't work:
  There's nothing to kill — it's already dead
  It's a record, not a process
```

### How `ps` Shows Process States

```
ps -eo pid,ppid,stat,comm

STAT column meanings:
  S    ← Sleeping (waiting for event)
  R    ← Running (on CPU right now)
  D    ← Uninterruptible sleep (usually disk I/O)
  T    ← Stopped (SIGSTOP or trace)
  Z    ← Zombie (defunct) ← what we hunt
  I    ← Idle kernel thread

Additional flags after main state:
  <    ← high priority (negative nice)
  N    ← low priority (positive nice)
  s    ← session leader
  l    ← multi-threaded
  +    ← in foreground process group

So: Sl = Sleeping, multi-threaded (typical daemon)
    Z  = Zombie (what we're hunting)
    S  = normal sleeping process (healthy)
```

### The Fix Logic — Why Killing Parent Works

```
Before killing parent:
  PID 873  (parent)  STAT=S  ← alive, has zombie children
  PID 874  (zombie)  STAT=Z  ← can't kill directly
  PID 876  (zombie)  STAT=Z
  PID 877  (zombie)  STAT=Z

After killing parent (SIGTERM → parent dies):
  PID 873 terminates
  Kernel runs cleanup:
    → checks if parent had children
    → those children (even zombies) get REPARENTED to init (PID 1)
    → init (systemd) automatically calls wait() on adopted zombies
    → zombie entries REMOVED from process table ✓

Final state:
  PID 874  GONE ✓
  PID 876  GONE ✓
  PID 877  GONE ✓
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "What is a zombie process? Why can't you kill it with `kill -9`?"

**Answer:**
> *"A zombie is a process that has finished executing but hasn't been removed from the process table because its parent hasn't called `wait()` to collect its exit status. It holds no memory, runs no code, consumes no CPU — it's just a row in the process table containing the exit code. `kill -9` sends SIGKILL to terminate a running process, but a zombie has no running code to kill — it's already dead. The only fix is to make the parent call `wait()`, either by fixing the parent application or by killing the parent, which triggers kernel reparenting to init, which then calls `wait()` automatically."*

---

### Q2. "How do you find zombie processes on a Linux system?"

**Answer — give all three methods:**
```bash
# Method 1 — awk on STAT column (most precise)
ps -eo pid,ppid,stat,comm | awk '$3=="Z"'

# Method 2 — grep for defunct keyword
ps aux | grep defunct | grep -v grep

# Method 3 — count only
ps -eo stat --no-headers | grep -c Z

# Method 4 — pgrep (if available)
pgrep -l -x 'Z'
```

---

### Q3. "How do you find which parent is responsible for zombie children?"

**Answer:**
> *"Every process has a PPID — Parent Process ID. I read it from the `ps` output and trace each zombie back to its parent. The `ps -eo pid,ppid,stat,comm` output gives me all four fields I need. Once I have the PPIDs, I look up those parent processes to understand what application is buggy and failing to call `wait()`."*

```bash
# Get zombie-to-parent mapping
ps -eo pid,ppid,stat,comm --no-headers | awk '$3=="Z"' | \
while read zpid par stat comm; do
    par_cmd=$(ps -p "$par" -o comm= 2>/dev/null)
    echo "Zombie $zpid (cmd: $comm) → Parent $par (cmd: $par_cmd)"
done
```

---

### Q4. "Why do zombies accumulate? What's the underlying bug in the application?"

**Answer:**
> *"The root cause is always the same: the parent process calls `fork()` to create children but never calls `wait()` or `waitpid()` to collect their exit status. This is a programming bug — the POSIX standard requires parents to reap their children. Common causes are: a multi-threaded application that forks but the thread handling `wait()` has a race condition, a signal handler that drops `SIGCHLD` events, or an application that forks workers and then crashes before calling `wait()`. The fix at the application level is to add a `SIGCHLD` handler that calls `waitpid(-1, NULL, WNOHANG)` in a loop."*

---

### Q5. "What's the maximum number of processes in the PID table and why do zombies matter?"

**Answer:**
> *"The PID limit is set by `/proc/sys/kernel/pid_max` — default 32,768 on 32-bit, up to 4 million on 64-bit. Each zombie consumes one PID slot. If zombies accumulate to the point of exhausting the PID table, `fork()` calls start failing with `EAGAIN` — the kernel can't allocate a new PID. This means the system can't spawn any new processes: no new SSH connections, no new commands, nothing. It's a hard system failure despite plenty of free memory and CPU. On Booking.com's scale with high-traffic servers, a buggy deployment that creates one zombie per request could exhaust the PID table within minutes."*

```bash
cat /proc/sys/kernel/pid_max    # check current limit
ps aux | wc -l                  # current process count
```

---

### Q6. "What's the difference between SIGTERM and SIGKILL when dealing with parent processes?"

**Answer:**
> *"For zombie cleanup, I try SIGTERM first — it gives the parent a chance to shut down gracefully, flush buffers, close connections, and importantly run its cleanup code which may call `wait()` on remaining children. If the parent is properly coded, SIGTERM triggers a clean exit that reaps zombies in the process. SIGKILL is the fallback when the parent is frozen or ignoring signals — the kernel forces termination immediately. Both result in the zombie children being adopted by init and reaped, but SIGTERM is safer for production because it doesn't interrupt in-flight work as abruptly."*

---

### Q7. "How would you prevent zombie accumulation in a production system long-term?"

**Answer — three layers:**

> **Layer 1 — Application fix (root cause):**
```c
// In the parent process, add SIGCHLD handler
signal(SIGCHLD, SIG_IGN);          // simple: ignore child exit → auto-reap
// OR
void handler(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0);  // reap all ready children
}
signal(SIGCHLD, handler);
```

> **Layer 2 — Monitoring (detect before crisis):**
```bash
# Alert when zombie count exceeds threshold
ZOMBIES=$(ps -eo stat --no-headers | grep -c Z)
[ "$ZOMBIES" -gt 10 ] && \
    echo "ALERT: $ZOMBIES zombies on $(hostname)" | mail ops@booking.com
```

> **Layer 3 — Systemd containment:**
```ini
# /etc/systemd/system/myapp.service
[Service]
# systemd kills entire cgroup on stop — no orphaned zombies
KillMode=control-group
# Set PID limit — prevents PID exhaustion even if zombies accumulate
TasksMax=500
```

---

### Q8. "What's the role of init (PID 1) in zombie cleanup?"

**Answer:**
> *"Init (PID 1 — systemd on modern Linux) is the ultimate parent of all orphaned processes. When any process dies, the kernel checks if its children are still alive. If yes, those children get reparented to init. Init has a built-in `wait()` loop that constantly reaps any children that finish, including adopted zombies. This is why killing the buggy parent works — even if the parent dies without calling `wait()`, init adopts the zombies and calls `wait()` on them automatically, clearing them from the process table. Init is the system's garbage collector for processes."*

---

## Part 5: Cheat Sheet

```
FIND ZOMBIES:
  ps aux | awk '$8=="Z"'                    # by STAT column
  ps aux | grep defunct | grep -v grep      # by keyword
  ps -eo pid,ppid,stat,comm | awk '$3=="Z"' # cleanest — PID+PPID
  ps -eo stat --no-headers | grep -c Z      # count only

MAP TO PARENTS:
  ps -eo pid,ppid,stat,comm | awk '$3=="Z" {print $2}' | sort -u
  # → gives PPID of each zombie's parent

KILL PARENT (the fix):
  kill -15 <PPID>   # SIGTERM first (graceful)
  kill -9  <PPID>   # SIGKILL if SIGTERM ignored

VERIFY CLEAN:
  ps -eo stat --no-headers | grep -c Z   # should be 0

WHAT Z MEANS IN ps STAT:
  Z = Zombie (defunct) — dead but not reaped
  < after STAT = high priority
  s = session leader

WHY kill -9 FAILS ON ZOMBIES:
  Zombie = already dead, no code running
  kill sends signal to RUNNING process
  Nothing to signal → nothing happens

ROOT CAUSE:
  Parent calls fork() but not wait()
  Fix in code: signal(SIGCHLD, SIG_IGN)
               or waitpid(-1, NULL, WNOHANG) in SIGCHLD handler

PID EXHAUSTION CHECK:
  cat /proc/sys/kernel/pid_max    # max PIDs (~32K default)
  ps aux | wc -l                  # current count
```

> **Booking.com interview tip:** They run massive concurrent workloads — millions of requests per second across thousands of processes. Mention PID table exhaustion as the real production risk, the `TasksMax=` systemd directive as containment, and `SIGCHLD + waitpid()` as the proper application-level fix. Showing you understand both the immediate fix (kill parent) and the root cause prevention (fix the `wait()` call) is what separates a senior SRE answer at this level.
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Monitoring Process Ownership
> **Company:** HashiCorp | **Difficulty:** Medium
---
#### **Scenario**
The server is consuming excessive resources. This server is used by multiple teams with their own credentials (e.g. each team has a username `dev-team`, `qa-team`, `ops-team` etc).
#### **Task**
Identify which user is running the most **number of processes** (count) on the server, regardless of CPU or memory usage, and write that username to: `/home/devops/solution.txt`
#### **Example**
A single username written to the expected output file.
```
<username>
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/monitoring-process-ownership)

## Monitoring Process Ownership — Full Deep Dive

---

## Part 1: Understand It Simply

### What Are We Finding?

```
Server running processes from multiple teams:

  dev-team  → 47 processes  ← MOST — this is the answer
  qa-team   → 23 processes
  ops-team  → 31 processes
  root      → 12 processes
  www-data  →  8 processes

Task: find "dev-team" → write to solution.txt
```

### The Pipeline Logic

```
ps aux          →  list every process with its owner
awk '{print $1}' →  extract just the username column
sort            →  group identical usernames together
uniq -c         →  count consecutive identical lines
sort -rn        →  sort by count, highest first
head -1         →  take the top line (most processes)
awk '{print $2}' →  extract just the username from "47 dev-team"
```

---

## Part 2: Build and Execute

### Step 1 — Simulate the Multi-Team EnvironmentThe processes are already running from earlier attempts. Now the full investigation.

### Step 2 — See the Full Process Landscape### Step 3 — The Core Investigation: Count Per User`dev-team` identified and saved. Now the deep explanation.

---

## Part 3: Every Concept Explained Simply

### The Full Pipeline — Every Stage Visualised

```bash
ps aux | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'
```

```
ps aux output:
  USER       PID  %CPU  ...
  root         1   0.0  ...   ← NR=1 (header) — SKIP
  root         2   0.0  ...   ← NR>1 — print $1 = "root"
  dev-team   746   0.0  ...   ← NR>1 — print $1 = "dev-team"
  dev-team   747   0.0  ...

After awk NR>1 {print $1}:
  root
  root
  dev-team
  dev-team
  dev-team
  ops-team
  qa-team

After sort:           After uniq -c:    After sort -rn:
  dev-team              12 dev-team       59 root
  dev-team               8 ops-team       12 dev-team
  dev-team               5 qa-team         8 ops-team
  dev-team              59 root            5 qa-team
  ops-team
  qa-team
  root
  root
  root

After head -1:        After awk {print $2}:
  59 root               root
```

---

### `uniq -c` — Why `sort` Must Come First

```
WITHOUT sort first:
  root          ← uniq sees: root
  dev-team      ← NEW value → new count: 1 dev-team
  root          ← NEW value → new count: 1 root  ← WRONG
  dev-team      ← NEW value → new count: 1 dev-team ← WRONG

uniq only collapses CONSECUTIVE identical lines
sort brings all identical lines together FIRST

WITH sort first:
  dev-team      ← uniq sees: dev-team (start count)
  dev-team      ← same → count 2
  dev-team      ← same → count 3 ... → "3 dev-team"
  ops-team      ← NEW → "2 ops-team"
  root          ← NEW → "59 root"
```

---

### `sort -rn` — Flags Explained

```
sort FLAGS:
  -r  →  reverse order (highest first, not lowest)
  -n  →  numeric sort (not lexicographic/alphabetical)

Without -n (alphabetical sort):
  "8 ops-team"
  "59 root"    ← "5" < "8" alphabetically — WRONG ORDER
  "12 dev-team"

With -n (numeric sort):
  "59 root"    ← 59 is largest ✓
  "12 dev-team"
  "8 ops-team"
  "5 qa-team"
```

---

### `awk 'NR>1'` — Skipping the Header Row

```
ps aux header:
  USER  PID  %CPU  %MEM  VSZ  RSS  TTY  STAT  START  TIME  COMMAND
  ↑
  If included, "USER" would appear in our count as a username
  NR = Number of Record (line number)
  NR>1 = skip line 1 (the header) ✓

Alternative ways to skip header:
  ps aux | tail -n +2          # skip first line via tail
  ps aux | grep -v "^USER"     # grep out header line
  ps -eo user --no-headers     # cleaner — no header at all ← best
```

---

### Alternative Approaches — Same Result

```bash
# Method 1 — classic pipeline (most readable)
ps aux | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'

# Method 2 — ps with no-headers (cleaner)
ps -eo user --no-headers | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'

# Method 3 — pure awk (no sort/uniq needed)
ps -eo user --no-headers | awk '{count[$1]++} END{
    for(u in count) if(count[u]>max){max=count[u]; top=u}
    print top
}'

# Method 4 — using /proc directly
ls -la /proc/[0-9]*/exe 2>/dev/null | awk '{print $3}' | ... (complex)
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How do you find which user owns the most processes?"

**Answer:**
> *"I use a five-stage pipeline. `ps aux` lists all processes. `awk 'NR>1 {print $1}'` extracts just the username column, skipping the header. `sort` groups identical usernames together since `uniq` only counts consecutive lines. `uniq -c` counts each group. `sort -rn` puts the highest count first. `head -1` takes the top entry. Then a final `awk '{print $2}'` strips the count leaving just the username."*

```bash
ps aux | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'
```

---

### Q2. "Why must `sort` come before `uniq -c`?"

**Answer:**
> *"`uniq` only collapses consecutive identical lines — it doesn't scan the whole file. Without sorting first, if `root` appears on lines 1, 5, and 10, `uniq` would count three separate groups of 1 instead of one group of 3. Sorting brings all identical usernames together into one consecutive block, so `uniq -c` can count the entire group correctly. This sort-then-uniq pattern is fundamental Linux text processing."*

---

### Q3. "What does `sort -rn` do? Why are both flags needed?"

**Answer:**
> *"`-n` means numeric sort — it interprets the leading number from `uniq -c` as an integer. Without it, sorting is lexicographic and `59` sorts before `8` because `'5' < '8'` alphabetically. `-r` reverses the order so the highest count comes first instead of last. Both flags together give descending numeric sort — exactly what we need to put the top user at the top."*

```bash
# Wrong (lexicographic): 59 → 8 → 5 → 12 (because '1' < '5' < '8')
sort -r

# Correct (numeric descending): 59 → 12 → 8 → 5
sort -rn
```

---

### Q4. "What's `NR>1` in awk and why is it needed?"

**Answer:**
> *"`NR` is awk's built-in variable for the current line number — Number of Record. `NR>1` means 'process this line only if it's not the first line', which skips the `ps aux` header row containing `USER PID %CPU ...`. Without it, `USER` would appear as a username in the output. The cleaner alternative is `ps -eo user --no-headers` which tells ps not to print the header at all."*

---

### Q5. "How would you exclude system users and only count team users?"

**Answer:**
> *"Two approaches. If team users are known, filter explicitly. If team users follow a naming convention — like containing a hyphen — filter by pattern:"*

```bash
# Exclude specific system users
ps aux | awk 'NR>1 && $1!="root" && $1!="www-data" && $1!="nobody" {print $1}' \
       | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'

# Filter by naming convention (team-name pattern)
ps aux | awk 'NR>1 && $1~/\-team$/ {print $1}' \
       | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'

# Exclude system UIDs (users with UID < 1000 are typically system)
ps -eo user,uid --no-headers | awk '$2>=1000 {print $1}' \
    | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'
```

---

### Q6. "How does the `ps -eo` format differ from `ps aux`?"

**Answer:**
> *"`ps aux` is a BSD-style shorthand that shows a fixed set of columns in a fixed order. `ps -eo` is POSIX-style and lets you specify exactly which columns you want — `user`, `pid`, `%cpu`, `%mem`, `rss`, `comm`, etc. For scripting, `-eo` is more reliable because the column positions don't change based on terminal width, and `--no-headers` cleanly removes the header line without needing `awk NR>1`."*

```bash
ps aux                          # fixed columns, has header
ps -eo user,pid,%cpu,comm       # choose your columns
ps -eo user --no-headers        # user only, no header ← cleanest for counting
```

---

### Q7. "How would you monitor process count per user continuously?"

**Answer:**
> *"For live monitoring I'd use `watch` to refresh the command every few seconds, or build a loop that alerts when a threshold is crossed:"*

```bash
# Live dashboard — refreshes every 2 seconds
watch -n 2 'ps -eo user --no-headers | sort | uniq -c | sort -rn | head -10'

# Alert when any user exceeds 50 processes
while true; do
    OFFENDER=$(ps -eo user --no-headers | sort | uniq -c | sort -rn | \
               awk '$1>50 {print $2; exit}')
    if [ -n "$OFFENDER" ]; then
        echo "ALERT: $OFFENDER has too many processes" | mail ops@hashicorp.com
    fi
    sleep 30
done

# Log to file for trending
echo "$(date),$(ps -eo user --no-headers | sort | uniq -c | sort -rn | head -1)" \
    >> /var/log/process_ownership.csv
```

---

### Q8. "How would you count processes per user using `/proc` directly instead of `ps`?"

**Answer:**
> *"`/proc` is the source of truth — `ps` just reads from it. Every running process has a directory at `/proc/<PID>/`. The file `/proc/<PID>/status` contains the `Uid:` field with the real UID. I can map UIDs to usernames with `getent passwd`:"*

```bash
# Read UIDs from /proc directly
for pid in /proc/[0-9]*/status; do
    awk '/^Uid:/{print $2}' "$pid" 2>/dev/null
done | sort | uniq -c | sort -rn | while read count uid; do
    user=$(getent passwd "$uid" | cut -d: -f1)
    echo "$count $user"
done | head -5
```

> *"This is more verbose but bypasses `ps` entirely — useful when `ps` is unavailable or when you need to audit processes including kernel threads."*

---

## Part 5: Cheat Sheet

```
CORE PIPELINE:
  ps aux | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'

CLEANER VERSION (no header issue):
  ps -eo user --no-headers | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'

FULL LEADERBOARD:
  ps -eo user --no-headers | sort | uniq -c | sort -rn

SAVE TO FILE:
  ps -eo user --no-headers | sort | uniq -c | sort -rn | \
    head -1 | awk '{print $2}' > /home/devops/solution.txt

EXCLUDE ROOT/SYSTEM:
  ps -eo user,uid --no-headers | awk '$2>=1000 {print $1}' | \
    sort | uniq -c | sort -rn | head -1 | awk '{print $2}'

KEY COMMAND FLAGS:
  ps aux            → all processes, BSD format
  ps -eo user       → custom columns, POSIX format
  --no-headers      → skip header row
  awk NR>1          → skip first line (header)
  sort              → group identical lines (required before uniq)
  uniq -c           → count consecutive identical lines
  sort -rn          → numeric descending sort
  head -1           → take first line (highest count)
  awk '{print $2}'  → extract second field (username)

LIVE MONITORING:
  watch -n 2 'ps -eo user --no-headers | sort | uniq -c | sort -rn'
```

> **HashiCorp interview tip:** They're a DevOps/infrastructure company — they'll appreciate mentioning `watch` for live monitoring and the idea of feeding this into a metrics system like Prometheus with a custom collector. Bonus points for knowing `/proc` as the source of truth behind `ps` — that shows kernel-level understanding that separates senior candidates.
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++


# Port Conflict Resolution
> **Company:** Datadog | **Difficulty:** Easy
---
#### **Scenario**
An application `/home/interview/server.sh` fails to start.
#### **Task**
Find the cause of failure and resolve it so the server can start successfully.
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/port-conflict-resolution)

## Port Conflict Resolution — Full Deep Dive

---

## Part 1: Understand It Simply

### What is a Port Conflict?

```
Every network service binds to a PORT — a numbered door on the server.
Only ONE process can own a port at a time.

Port 8080 is like a physical office room:
  If someone is already in room 8080 →
  the next person trying to enter → BLOCKED

Application tries to start:
  server.sh → "I need port 8080"
  OS checks  → "port 8080 already taken by PID 1234"
  OS replies → "Address already in use" ✗
  server.sh  → crashes/fails to start
```

### The Investigation Flow

```
server.sh fails to start
        ↓
Step 1: Run it — read the error message
        ↓
Step 2: Find which port it needs
        ↓
Step 3: Find which process owns that port (ss / netstat / lsof)
        ↓
Step 4: Kill the occupying process
        ↓
Step 5: Start server.sh — verify it works
```

---

## Part 2: Build and Execute

### Step 1 — Create the Environment### Step 2 — Reproduce the FailureFailure confirmed. Now the full investigation.

### Step 3 — Diagnose: Find What Owns Port 2024### Step 4 — Fix: Update server.sh to Free Port + VerifyServer starts cleanly. Now the deep explanation.

---

## Part 3: Every Concept Explained Simply

### How Ports Work

```
Server has ONE IP address: 192.168.1.10
But runs MANY services simultaneously.
Ports (1–65535) are numbered "channels":

  192.168.1.10:22    ← SSH daemon
  192.168.1.10:80    ← nginx (HTTP)
  192.168.1.10:443   ← nginx (HTTPS)
  192.168.1.10:5432  ← PostgreSQL
  192.168.1.10:2024  ← process_api ← OCCUPIED
  192.168.1.10:3000  ← your app ← FREE ✓

OS rule: ONE process per port (unless SO_REUSEPORT set)
Violation → "EADDRINUSE: Address already in use"
```

### `/proc/net/tcp` — Reading Without Tools

```
cat /proc/net/tcp
  sl  local_address  rem_address  st ...  inode
   0: 00000000:07E8  00000000:0000  0A ...  600

Decode:
  local_address = 00000000:07E8
    00000000  → IP 0.0.0.0 (all interfaces)
    07E8      → port in hex → 0x07E8 = 2024

  st = 0A = 10 decimal = TCP_LISTEN state
  (0A = listening, 01 = established, 06 = TIME_WAIT)

  inode = 600  → socket inode number
```

### Mapping Inode → PID (No `lsof` Needed)

```
Every process has open file descriptors at:
  /proc/<PID>/fd/

Sockets show up as:
  /proc/1/fd/10 → socket:[600]
                           ↑
                     inode 600 = port 2024

So the lookup is:
  1. Get inode from /proc/net/tcp
  2. Walk /proc/[0-9]*/fd/*
  3. readlink each fd
  4. Match "socket:[inode]"
  5. Extract PID from path → found the owner
```

### The Two Types of Fix

```
FIX TYPE 1: Kill the occupier
  When: rogue/unknown process is occupying the port
  When: a crashed service left a stale socket
  How:
    kill -15 <PID>  → graceful
    kill -9  <PID>  → force

FIX TYPE 2: Change your app's port (our case)
  When: occupier is a legitimate system process (can't/shouldn't kill)
  When: occupier is another critical service
  How:
    Edit server.sh / app config / env var → use different port

Rule: identify WHAT owns the port before deciding which fix
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How do you find which process is using a specific port?"

**Answer — give all three methods, ranked:**

```bash
# Method 1 — ss (modern, fastest)
ss -tlnp | grep :8080
# LISTEN  0  5  0.0.0.0:8080  *  users:(("python3",pid=1234,fd=3))

# Method 2 — lsof (most detailed)
lsof -i :8080
# COMMAND  PID USER  FD  TYPE  NODE NAME
# python3 1234 root  3u  IPv4  600  TCP *:8080 (LISTEN)

# Method 3 — netstat (legacy, still common)
netstat -tlnp | grep :8080

# Method 4 — /proc/net/tcp (no tools needed, always available)
awk 'NR>1 {split($2,a,":"); if(a[2]=="1F90") print $10}' /proc/net/tcp
# → get inode, then: grep -r "socket:\[inode\]" /proc/*/fd/
```

---

### Q2. "What does 'Address already in use' mean and what causes it?"

**Answer:**
> *"It's the POSIX error `EADDRINUSE` — the application called `bind()` on a port that's already registered to another process. Only one socket can own a port at a time (unless `SO_REUSEPORT` is set). Common causes are: another instance of the same application is already running, a previous crash left the socket in `TIME_WAIT` state (briefly holds the port after close), a completely different service was configured to use the same port, or systemd/the OS has a service pre-bound to that port."*

---

### Q3. "What's `ss` and how does it differ from `netstat`?"

**Answer:**
> *"`ss` (Socket Statistics) is the modern replacement for `netstat` — it reads directly from kernel socket structures via netlink, making it significantly faster on servers with thousands of connections. `netstat` reads `/proc/net/tcp` which is slower. Both show the same data — listening ports, established connections, owning PIDs. Key flags for port investigation are `-t` (TCP), `-l` (listening only), `-n` (numeric, no DNS lookup), `-p` (show process)."*

```bash
ss -tlnp    # TCP Listening Numeric with Process — the go-to
ss -tulnp   # add -u for UDP too
ss -s       # summary statistics
```

---

### Q4. "How do you decode `/proc/net/tcp` manually? What is the hex format?"

**Answer:**
> *"The `local_address` column is `IP:PORT` both in hex. The IP is in little-endian hex — `0100007F` is `127.0.0.1`. The port is plain hex — `1F90` is `0x1F90 = 8080`. The state column uses hex codes: `0A` = `LISTEN`, `01` = `ESTABLISHED`, `06` = `TIME_WAIT`. The inode column links to `/proc/<PID>/fd/` entries — any fd that `readlink` returns as `socket:[inode]` belongs to the process owning that port."*

```bash
# Decode port from hex:
python3 -c "print(int('1F90', 16))"  # 8080
# Encode port to hex for searching:
python3 -c "print(hex(8080).upper().lstrip('0X').zfill(4))"  # 1F90
```

---

### Q5. "When would you kill the occupying process vs change your app's port?"

**Answer:**
> *"Kill the occupier when: it's an unknown or rogue process, a zombie socket from a crashed service, a duplicate instance of the same app. Change your app's port when: the occupier is a legitimate system service (SSH on 22, HTTP on 80), it's a third-party service you can't control, or the port conflict is in a development environment where any free port works. At Datadog scale, the production answer is usually environment variable or config file driven port assignment — never hardcode ports, so changing `PORT=8080` in the config restarts cleanly without touching the binary."*

---

### Q6. "What is `TIME_WAIT` and why does it cause port conflicts after a crash?"

**Answer:**
> *"`TIME_WAIT` is a TCP state a socket enters after closing. It holds the port for `2 × MSL` (Maximum Segment Lifetime — typically 60-120 seconds) to absorb any delayed packets still in transit. If a server crashes and restarts immediately, its port may still be in `TIME_WAIT` from the previous connection, causing `EADDRINUSE`. The fix for servers is `SO_REUSEADDR` socket option — it allows binding to a port in `TIME_WAIT`. Most web frameworks set this automatically. You can see `TIME_WAIT` sockets with `ss -t state time-wait`."*

---

### Q7. "How would you make port conflict detection part of a startup script?"

**Answer:**
```bash
#!/bin/bash
PORT=8080

# Check if port is free before starting
check_port() {
    local port=$1
    local hex=$(printf '%04X' $port)
    grep -q ":${hex} " /proc/net/tcp 2>/dev/null && return 1  # in use
    return 0  # free
}

if ! check_port $PORT; then
    # Find who owns it
    HEX=$(printf '%04X' $PORT)
    INODE=$(awk -v h=":$HEX " '$0~h {print $10}' /proc/net/tcp | head -1)
    OWNER_PID=$(grep -rl "socket:\[$INODE\]" /proc/*/fd/ 2>/dev/null | \
                head -1 | cut -d/ -f3)
    OWNER_CMD=$(cat /proc/$OWNER_PID/comm 2>/dev/null)
    echo "ERROR: Port $PORT occupied by PID $OWNER_PID ($OWNER_CMD)"
    echo "Run: kill -15 $OWNER_PID  to free the port"
    exit 1
fi

exec /home/interview/server.sh
```

---

## Part 5: Cheat Sheet

```
FIND WHO OWNS A PORT:
  ss -tlnp | grep :<port>            # modern (fastest)
  lsof -i :<port>                    # most detailed
  netstat -tlnp | grep :<port>       # legacy
  fuser <port>/tcp                   # simple PID output
  cat /proc/net/tcp | grep <HEX>     # no tools needed

CONVERT PORT TO HEX:
  python3 -c "print(hex(8080).upper()[2:].zfill(4))"  # → 1F90
  printf '%04X\n' 8080                                  # → 1F90

STATE CODES IN /proc/net/tcp:
  0A = LISTEN      01 = ESTABLISHED
  06 = TIME_WAIT   03 = SYN_RECV

KILL THE OCCUPIER:
  kill -15 <PID>   # graceful first
  kill -9  <PID>   # force if needed

OR CHANGE YOUR PORT:
  sed -i 's/PORT=8080/PORT=3000/' server.sh   # edit config
  PORT=3000 ./server.sh                        # env override

FIND A FREE PORT:
  ss -tlnp | awk '{print $4}' | cut -d: -f2 | sort -n  # used
  python3 -c "
    import socket
    s = socket.socket()
    s.bind(('', 0))
    print(s.getsockname()[1])  # OS assigns free port
    s.close()"

CHECK TIME_WAIT SOCKETS:
  ss -t state time-wait | grep <port>
```

> **Datadog interview tip:** They're an observability company — they'll love if you mention instrumenting port conflicts as a metric (`EADDRINUSE` errors in application logs → alert before it causes an outage), and the `SO_REUSEADDR`/`SO_REUSEPORT` distinction for zero-downtime restarts. That's the kind of systematic thinking that makes Datadog's infrastructure reliable at scale.
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
# Purge Empty Folders
> **Company:** CrowdStrike | **Difficulty:** Easy
---
#### **Scenario**
The `/tmp` directory has accumulated numerous leftover folders from previous application runs and temporary scripts. Many of these directories are now empty and can be safely removed.
#### **Task**
Write a command or short script that searches through `/tmp`, finds all empty directories recursively, and deletes them without affecting any directories that contain files or subdirectories.
#### **Example**
```
# Before (empty directories present)
/tmp/old_build_cache/
/tmp/session_temp_1234/
/tmp/extract_workspace/
/tmp/app_tmp_5678/
/tmp/active_project/file.txt
```
```
# After (empty directories removed)
/tmp/active_project/file.txt
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/purge-empty-folders)


# Rapid Disk Growth on Var
> **Company:** Google | **Difficulty:** Hard
---
#### **Scenario**
Disk usage on the `/var` partition is at 92% and increasing rapidly. You need to identify the largest files consuming space and determine if they're actively used by processes or need log rotation.
#### **Task**
Find the 10 largest files under `/var` and save them to `/home/devops/largest_var_files.txt`, check which processes are using these files and save results to `/home/devops/file_processes.txt`, and verify log rotation configuration for any log files found, saving results to `/home/devops/logrotate_status.txt`.
#### **Example**
```
# File: /home/devops/largest_var_files.txt
2.3G    /var/log/mysql/mysql-slow.log
1.8G    /var/lib/docker/overlay2/abc123/diff/app/data.db
1.3G    /var/log/nginx/access.log
891M    /var/cache/apt/archives/linux-image-generic.deb
655M    /var/log/syslog.1
450M    /var/log/myapp/app.log
...
```
```
# File: /home/devops/file_processes.txt
tail     44 root    3r   ... /var/log/mysql/sorted.log
tail     45 root    3r   ... /var/log/nginx/test.log
tail     46 root    3r   ... /var/log/hello.1
...
```
```
# File: /home/devops/logrotate_status.txt
/etc/logrotate.d/test:/var/log/test.log
/etc/logrotate.d/test:/var/log/hello.log
...
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/rapid-disk-growth-on-var)



# Real-Time Log Timestamping
> **Company:** Adobe | **Difficulty:** Medium
---
#### **Scenario**
You're troubleshooting a service that produces untagged log output when run manually, making it difficult to analyze timing and sequence of events.
#### **Task**
Create a command that reads from standard input line by line and appends the current timestamp to the end of each line as it's read. Test it interactively by piping output to verify it works, then save the solution to a shell script at `/usr/local/bin/timestamp.sh` and make it executable so it can be used in any pipeline.
#### **Example**
```
# Before (untagged log output)
Application started
Processing request #1234
Database connection established
Request completed
```
```
# After (timestamped in real-time)
Application started - 2025-11-06 15:30:45
Processing request #1234 - 2025-11-06 15:30:46
Database connection established - 2025-11-06 15:30:47
Request completed - 2025-11-06 15:30:48
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/real-time-log-timestamping)

# Three Tasks — Complete Deep Dive

---

## TASK 1: Purge Empty Folders (CrowdStrike | Easy)

### Understand It Simply

```
/tmp/
├── old_build_cache/          ← EMPTY → DELETE ✓
├── session_temp_1234/        ← EMPTY → DELETE ✓
│   └── nested_empty/         ← EMPTY → DELETE ✓ (child first)
├── extract_workspace/        ← EMPTY → DELETE ✓
├── app_tmp_5678/             ← EMPTY → DELETE ✓
└── active_project/
    └── file.txt              ← HAS FILE → KEEP ✓

Rule: only delete directories with ZERO contents
      (no files AND no subdirectories inside)
```**One command. Zero empty directories. Non-empty dirs untouched.**

---

### Part 2: Concepts + Interview Q&A

**Key concept — `find -empty` + `-delete` ordering:**

```
find processes DEEPEST paths first with -delete

Directory tree:           find output order:
/tmp/session/             3. /tmp/session/       ← now empty → deleted
  └── nested_empty/       1. /tmp/session/nested_empty ← deleted first

If order were reversed (parent first):
  Delete /tmp/session/ ← ERROR: not empty, still has nested_empty/
```

**`-mindepth 1` — the safety flag:**

```bash
find /tmp -type d -empty -delete        # DANGEROUS — could delete /tmp itself!
find /tmp -mindepth 1 -type d -empty -delete  # safe — skips /tmp root ✓
```

---

**Interview Q&A:**

**Q: "What's the single command to delete all empty directories in /tmp recursively?"**
> `find /tmp -mindepth 1 -type d -empty -delete`

**Q: "Why does `-delete` handle nested empty directories without extra logic?"**
> *"find traverses depth-first, so it always encounters a leaf directory before its parent. With `-delete`, the deepest empty directory gets deleted first, making its parent potentially empty, which then also matches `-empty` and gets deleted in the same pass — cascading upward automatically."*

**Q: "How would you do a dry run first?"**
```bash
# Preview only — no deletion
find /tmp -mindepth 1 -type d -empty -print

# Then execute:
find /tmp -mindepth 1 -type d -empty -delete
```

**Q: "What's the risk if you forget `-mindepth 1`?"**
> *"Without `-mindepth 1`, if `/tmp` itself were somehow empty (or if you ran this on a different path like a freshly created temp dir), find could match and delete the root directory you're searching from. `-mindepth 1` ensures the start directory is never touched."*

---

---

## TASK 2: Rapid Disk Growth on /var (Google | Hard)

### Understand It Simply

```
/var is at 92% — three questions to answer:
  1. WHAT files are largest?              → largest_var_files.txt
  2. ARE they being actively used?        → file_processes.txt
                                            (open file handles = actively written)
  3. IS log rotation configured for them? → logrotate_status.txt
```---

### Concepts + Interview Q&A (Task 2)

**Why `find -type f -exec du -h {} +` beats `du -ah`:**

```bash
du -ah /var | sort -rh | head -10
# Problem: includes DIRECTORIES in results (noisy)
# /var      200G  ← directory total (misleading)
# /var/log   50G  ← also a directory

find /var -type f -exec du -h {} + | sort -rh | head -10
# Cleaner: FILES ONLY
# /var/log/mysql/mysql-slow.log   2.3G ← actual file ✓
```

**Open file handles — why they matter:**

```
If a process has a file open and you delete it:
  ls /var/log/app.log  →  GONE (directory entry removed)
  BUT the process still writes to it via its fd
  The disk space is NOT freed until the process closes the fd
  df -h still shows the space consumed!

Fix:
  kill/restart the process  →  fd closed  →  space freed
  OR
  truncate file while process runs:
  > /var/log/app.log      (truncate to zero, process keeps fd) ✓
  cat /dev/null > /var/log/app.log  (same thing)
```

**Interview Q&A:**

**Q: "How do you find the 10 largest files in a directory?"**
```bash
find /var -type f -exec du -h {} + 2>/dev/null | sort -rh | head -10
# Alternative (faster on large trees):
find /var -type f -printf "%s\t%p\n" | sort -rn | head -10 | \
  awk '{printf "%.1fMB\t%s\n", $1/1048576, $2}'
```

**Q: "Why might `df -h` still show disk full even after deleting a large log file?"**
> *"If a process still has an open file descriptor to the deleted file, the kernel keeps the inode and data blocks allocated until that fd is closed. The directory entry is gone (file not visible in `ls`) but the space isn't freed. You find these with `lsof | grep deleted` or checking `/proc/*/fd` for links to `(deleted)` files. Fix: restart the process or truncate the file with `> /path/to/log` while it's still open."*

**Q: "What's `sort -rh` and why does it handle sizes like `2.3G`, `891M` correctly?"**
> *"`-h` is human-readable sort — it understands size suffixes (K, M, G, T) and sorts them correctly so 2.3G appears above 891M. Without `-h`, alphabetical sort would put `891M` before `2.3G` because `8` > `2`. `-r` reverses the order so largest appears first."*

---

---

## TASK 3: Real-Time Log Timestamping (Adobe | Medium)

### Understand It Simply

```
Service outputs logs with NO timestamps:
  Application started
  Processing request #1234

We need:
  Application started - 2025-11-06 15:30:45
  Processing request #1234 - 2025-11-06 15:30:46

Key challenge: "real-time" — each line gets the time
               IT arrives, not the time the script started
```Timestamps differ by 2 seconds each — **genuinely real-time per line.**Test 2 shows `03:57:36`, `03:57:37`, `03:57:38` — one second apart. **Truly real-time.**

---

### Concepts + Interview Q&A (Task 3)

**Why `IFS= read -r` matters:**

```bash
while IFS= read -r line; do
        ↑              ↑
        │              -r = don't interpret backslashes
        │              (without -r: "hello\nworld" → "hellonworld")
        │
        IFS= = don't trim leading/trailing whitespace
        (without IFS=: "  indented line" → "indented line")
```

**`fflush()` in awk — the buffering problem:**

```
Without fflush():
  awk collects output in a buffer (typically 4KB or 8KB)
  Output appears in BURSTS, not line by line
  Defeats the purpose of real-time timestamping

With fflush():
  awk flushes output buffer after every line
  Each line appears immediately as it's processed
  True real-time behaviour ✓

awk '{ print $0 " - " strftime("%H:%M:%S"); fflush() }'
                                                  ↑
                                          flush after each line
```

**`strftime` vs `system("date")`:**

```bash
# BAD — spawns a new process for EVERY line (slow)
awk '{ system("date +%H:%M:%S"); print $0 }'

# GOOD — strftime is built into awk (fast, no subprocess)
awk '{ print $0 " - " strftime("%Y-%m-%d %H:%M:%S") }'
```

---

**Interview Q&A:**

**Q: "Explain `while IFS= read -r line`. Why each part?"**
> *"`while` loops until stdin is exhausted. `read` reads one line at a time. `-r` disables backslash interpretation — without it, `\n` in a log line would become a literal newline. `IFS=` (empty) prevents read from stripping leading and trailing whitespace — critical for indented log output or YAML-formatted logs. Together they preserve the line exactly as received."*

**Q: "What's the difference between `awk strftime` and calling `date` in a subshell?"**
> *"`awk strftime` calls a C function internally — no subprocess created, runs in microseconds. `$(date +...)` spawns a new shell and date process for every single line — on a high-throughput service producing 10,000 lines/second, that's 10,000 fork-exec operations per second, adding significant overhead. For production log pipelines, always use `awk` with `strftime` and `fflush()`."*

**Q: "How would you prepend the timestamp instead of appending it?"**
```bash
# Append (task requirement):
awk '{ print $0 " - " strftime("%Y-%m-%d %H:%M:%S"); fflush() }'

# Prepend (common alternative):
awk '{ print strftime("%Y-%m-%d %H:%M:%S") " " $0; fflush() }'

# ISO 8601 format:
awk '{ print strftime("%Y-%m-%dT%H:%M:%S%z") " " $0; fflush() }'

# Millisecond precision (requires GNU awk):
awk 'BEGIN{cmd="date +%s%3N"}
     { cmd | getline ms; close(cmd); print $0 " - " ms; fflush() }'
```

---

## Master Cheat Sheet — All Three Tasks

```
TASK 1: PURGE EMPTY DIRS (CrowdStrike)
  find /tmp -mindepth 1 -type d -empty -delete
  # -mindepth 1  → don't delete /tmp itself
  # -type d      → directories only
  # -empty       → zero contents (no files, no subdirs)
  # -delete      → deepest-first, auto-cascades ✓
  # DRY RUN: replace -delete with -print

TASK 2: DISK GROWTH (Google)
  # Largest files:
  find /var -type f -exec du -h {} + | sort -rh | head -10

  # Open file handles:
  for pid in /proc/[0-9]*/fd; do
    for fd in $pid/*; do
      link=$(readlink $fd 2>/dev/null)
      [[ $link == /var/* ]] && echo "$link"
    done
  done

  # Logrotate check:
  grep -rl "/var/log/file.log" /etc/logrotate.d/

  # Deleted but held open (space not freed):
  # lsof | grep deleted
  # Fix: restart process OR: > /path/to/log (truncate)

TASK 3: TIMESTAMPING (Adobe)
  # One-liner:
  command | awk '{ print $0 " - " strftime("%Y-%m-%d %H:%M:%S"); fflush() }'

  # Script (stdin-safe):
  while IFS= read -r line; do
    echo "${line} - $(date '+%Y-%m-%d %H:%M:%S')"
  done

  # KEY: IFS= preserves whitespace, -r preserves backslashes
  # KEY: fflush() prevents buffering in awk
  # KEY: strftime() is faster than $(date) — no subprocess
```
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
# Rapid Disk Growth on Var
> **Company:** Google | **Difficulty:** Hard
---
#### **Scenario**
Disk usage on the `/var` partition is at 92% and increasing rapidly. You need to identify the largest files consuming space and determine if they're actively used by processes or need log rotation.
#### **Task**
Find the 10 largest files under `/var` and save them to `/home/devops/largest_var_files.txt`, check which processes are using these files and save results to `/home/devops/file_processes.txt`, and verify log rotation configuration for any log files found, saving results to `/home/devops/logrotate_status.txt`.
#### **Example**
```
# File: /home/devops/largest_var_files.txt
2.3G    /var/log/mysql/mysql-slow.log
1.8G    /var/lib/docker/overlay2/abc123/diff/app/data.db
1.3G    /var/log/nginx/access.log
891M    /var/cache/apt/archives/linux-image-generic.deb
655M    /var/log/syslog.1
450M    /var/log/myapp/app.log
...
```
```
# File: /home/devops/file_processes.txt
tail     44 root    3r   ... /var/log/mysql/sorted.log
tail     45 root    3r   ... /var/log/nginx/test.log
tail     46 root    3r   ... /var/log/hello.1
...
```
```
# File: /home/devops/logrotate_status.txt
/etc/logrotate.d/test:/var/log/test.log
/etc/logrotate.d/test:/var/log/hello.log
...
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/rapid-disk-growth-on-var)


## Rapid Disk Growth on /var — Full Deep Dive

---

## Part 1: Understand It Simply

### The Three Investigation Questions

```
/var is at 92% and growing fast
         ↓
Three parallel questions:

Q1: WHAT is consuming space?
    → find 10 largest files
    → largest_var_files.txt

Q2: WHO is writing to them right now?
    → find processes with open file handles
    → file_processes.txt
    → WHY: deleting a file held open = space NOT freed until process dies

Q3: ARE logs being rotated?
    → check logrotate config for each .log file found
    → logrotate_status.txt
    → WHY: unconfigured logs grow forever, no automatic cleanup
```

### The "Deleted But Held Open" Trap

```
You delete /var/log/app.log  →  it disappears from ls ✓
But process still has fd open →  kernel keeps data blocks allocated
df -h STILL shows disk full   →  space not freed! ← the trap

Solution: find open handles FIRST, then decide:
  Option A: kill/restart the process → fd closes → space freed
  Option B: truncate while open:  > /var/log/app.log ← safest
```

---

## Part 2: Build and Execute

### Step 1 — Create Realistic Environment### Step 2 — Task 1: Find the 10 Largest Files### Step 3 — Task 2: Find Which Processes Have Files OpenFile holders confirmed. Now the proper scan.

### Step 4 — Proper File Process Scan### Step 5 — Task 3: Verify Logrotate ConfigurationAll three files complete. Now the deep explanation.

---

## Part 3: Every Concept Explained Simply

### `du` vs `ls -lh` — Why `du` for Disk Usage

```bash
ls -lh /var/log/mysql-slow.log    # shows apparent size (bytes in file)
du -h  /var/log/mysql-slow.log    # shows DISK USAGE (blocks allocated)

Why they differ:
  Sparse files: file claims 2GB but only 100MB actually written
    ls -lh → 2.0G  (apparent)
    du -h  → 100M  (actual disk used)

  For disk growth investigation → du is correct ✓
  For file content analysis    → ls -lh is fine
```

### `find -exec du -h {} +` vs `du -ah`

```bash
# Option 1: du -ah /var
du -ah /var 2>/dev/null | sort -rh | head -10
# Problem: includes DIRECTORY totals in output
# /var           200G   ← misleading directory sum
# /var/log        50G   ← another directory sum
# /var/log/mysql   2G   ← actual file ← hard to find

# Option 2: find -type f (files ONLY)
find /var -type f -exec du -h {} + 2>/dev/null | sort -rh | head -10
# Clean: only actual files, no directory noise
# 30M   /var/lib/apt/lists/file.lz4  ← file ✓
# 15M   /var/log/mysql/mysql-slow.log ← file ✓
```

### The `/proc/PID/fd/` Open Handle Mechanism

```
Every running process has a directory:
  /proc/<PID>/
    ├── comm          ← process name
    ├── status        ← uid, gid, memory stats
    ├── cmdline       ← full command line
    └── fd/           ← ALL open file descriptors
         ├── 0 → /dev/null          (stdin)
         ├── 1 → /dev/pts/0         (stdout)
         ├── 2 → /dev/pts/0         (stderr)
         ├── 3 → /var/log/mysql-slow.log  ← FOUND IT
         └── 4 → socket:[12345]

readlink /proc/1154/fd/3
→ /var/log/mysql/mysql-slow.log   ← this is our file

This is exactly what lsof does internally:
  lsof = walks /proc/*/fd/ + formats nicely
  Our script = same thing, no dependency
```

### The Deleted-But-Open Trap — Full Mechanics

```
ls /proc/1154/fd/3 → (deleted)   ← file was rm'd but PID 1154 still writes
                                     space not freed until process dies

Proof:
  rm /var/log/app.log             ← gone from filesystem
  df -h still shows disk full     ← inode/blocks still allocated

Fixes:
  Fix 1: kill/restart the process  → fd closed → space freed immediately
  Fix 2: truncate while open:
    > /var/log/app.log             ← empties file, process keeps fd ✓
    truncate -s 0 /var/log/app.log ← same thing, more explicit

How to find these phantom files:
  find /proc/*/fd -ls 2>/dev/null | grep "(deleted)"
  OR
  ls -la /proc/*/fd 2>/dev/null | grep " (deleted)"
```

### Logrotate — How It Actually Works

```
/etc/logrotate.conf        ← main config
/etc/logrotate.d/nginx     ← per-app drop-in config

Config structure:
  /var/log/nginx/access.log {
      daily              ← rotate every day
      rotate 14          ← keep 14 rotated copies
      compress           ← gzip old copies
      delaycompress      ← don't compress most recent (process may still write)
      missingok          ← don't error if file missing
      notifempty         ← skip if file is empty
      postrotate
          nginx -s reopen  ← signal nginx to close old fd, open new file
      endscript
  }

What rotate does:
  access.log    → access.log.1
  access.log.1  → access.log.2.gz
  access.log.2.gz → access.log.3.gz
  ... up to rotate N
  access.log.14.gz → DELETED

WITHOUT logrotate:
  access.log grows forever → fills /var → outage
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How do you find the 10 largest files in a directory tree?"

**Answer:**
```bash
# Files only (cleanest):
find /var -type f -exec du -h {} + 2>/dev/null | sort -rh | head -10

# Alternative — faster on huge trees:
find /var -type f -printf "%s\t%p\n" | sort -rn | head -10 | \
  awk '{printf "%.0fMB\t%s\n", $1/1048576, $2}'

# du-based (includes directory totals — less precise):
du -ah /var 2>/dev/null | sort -rh | head -10
```

> *"I use `find -type f` to exclude directory entries — `du -ah` includes directory totals which inflate the results. `sort -rh` handles human-readable sizes correctly so `2.3G` sorts above `891M`."*

---

### Q2. "Why might deleting a large log file not free up disk space?"

**Answer:**
> *"If a process still has an open file descriptor to that file, the kernel keeps the inode and data blocks allocated even though the directory entry is gone. `df` still shows the space consumed. You can spot these phantom files with `find /proc/*/fd -ls 2>/dev/null | grep deleted`. The fix is either restart the process to close the fd, or truncate the file while it's open using `> /path/to/file` — this zeros the content while the process keeps writing to the same fd without errors."*

```bash
# Find deleted-but-held-open files:
find /proc/*/fd -ls 2>/dev/null | grep "(deleted)"

# Safe truncate without killing the process:
> /var/log/mysql/mysql-slow.log
# OR
truncate -s 0 /var/log/mysql/mysql-slow.log
```

---

### Q3. "How do you find which process has a specific file open without `lsof`?"

**Answer:**
> *"Walk `/proc/[0-9]*/fd/` — every process's open file descriptors are symlinks in that directory. `readlink` on each fd gives the real path. Match against the target file. Read `/proc/<PID>/comm` for the process name and `/proc/<PID>/status` for the UID. This is exactly what `lsof` does internally — it's just a `/proc` reader with nice formatting."*

```bash
for pid in $(ls /proc | grep '^[0-9]'); do
    for fd in /proc/$pid/fd/*; do
        target=$(readlink "$fd" 2>/dev/null)
        [ "$target" = "/var/log/mysql/mysql-slow.log" ] && \
            echo "PID $pid ($(cat /proc/$pid/comm 2>/dev/null)) has it open"
    done
done
```

---

### Q4. "What is `sort -rh` and why does it correctly sort sizes like `2.3G` and `891M`?"

**Answer:**
> *"`-h` is human-readable sort — it understands size suffixes. Without it, alphabetical sort treats `2.3G` and `891M` as strings: `'8' > '2'` so 891M sorts above 2.3G, which is wrong. With `-h`, the kernel compares magnitudes: G > M > K > nothing, then numeric within the same unit. `-r` reverses to get descending order. This combination is essential for any storage investigation script."*

```
Without -h:    891M, 655M, 2.3G, 1.8G   ← WRONG (string sort)
With -rh:      2.3G, 1.8G, 891M, 655M   ← CORRECT (numeric sort)
```

---

### Q5. "How does logrotate work and what happens when a log file has no rotation config?"

**Answer:**
> *"Logrotate is typically run daily by cron or systemd timer. It reads `/etc/logrotate.conf` and all files in `/etc/logrotate.d/`. For each configured log, it renames the current file (access.log → access.log.1), optionally compresses older copies, and deletes copies beyond the `rotate N` count. Critically, for processes that keep logs open, logrotate uses `copytruncate` (copy then zero the original) or sends a signal (`postrotate` block) to make the process reopen the file. Without any logrotate config, the log file grows indefinitely — the only limit is disk capacity."*

---

### Q6. "What's `copytruncate` and why would you use it over the default rotate?"

**Answer:**
> *"Default rotation: logrotate renames `app.log` to `app.log.1`, creates a new empty `app.log`, then signals the application to reopen its log fd. The app must support signals or have a reload mechanism. `copytruncate`: logrotate copies `app.log` to `app.log.1`, then truncates the original `app.log` to zero bytes — the process never needs to close/reopen. Use `copytruncate` when the application has no reload signal or can't be restarted (legacy apps). The downside: brief window where log lines written between copy and truncate are lost."*

```ini
/var/log/myapp/app.log {
    daily
    rotate 7
    copytruncate    ← safer for apps without reload support
    compress
}
```

---

### Q7. "How would you force logrotate to run immediately without waiting for cron?"

**Answer:**
```bash
# Debug mode — shows what WOULD happen (dry run):
logrotate -d /etc/logrotate.d/nginx

# Force rotation NOW (ignores lastrun timestamp):
logrotate -f /etc/logrotate.d/nginx
# OR force ALL configs:
logrotate -f /etc/logrotate.conf

# Verbose — see exactly what it does:
logrotate -vf /etc/logrotate.d/nginx

# Check last rotation times:
cat /var/lib/logrotate/status
```

---

### Q8. "System is at 92% disk — what's your systematic immediate action plan?"

**Answer — the Google-level answer with layers:**

> **Layer 1 — Immediate triage (< 2 minutes):**
```bash
df -h                           # confirm which partition is full
df -i                           # also check inodes — separate issue
```

> **Layer 2 — Find the offenders (< 5 minutes):**
```bash
find /var -type f -exec du -h {} + 2>/dev/null | sort -rh | head -10
du -sh /var/log/* | sort -rh | head -5   # quick directory drill-down
```

> **Layer 3 — Check for phantom held-open files:**
```bash
find /proc/*/fd -ls 2>/dev/null | grep "(deleted)"
```

> **Layer 4 — Immediate space recovery options (safest first):**
```bash
journalctl --vacuum-size=500M     # trim systemd journal
apt-get clean                     # clear apt cache
find /var/log -name "*.gz" -mtime +30 -delete  # old compressed logs
> /var/log/bigfile.log            # truncate held-open log
```

> **Layer 5 — Prevent recurrence:**
```bash
logrotate -f /etc/logrotate.conf  # force rotation now
# Add missing logrotate config for uncovered logs
# Set up monitoring: alert at 80% before hitting 92%
```

---

## Part 5: Cheat Sheet

```
FIND LARGEST FILES:
  find /var -type f -exec du -h {} + 2>/dev/null | sort -rh | head -10
  du -ah /var | sort -rh | head -10          # includes dirs (noisier)
  find /var -type f -printf "%s\t%p\n" | sort -rn | head -10  # bytes

FIND OPEN FILE HANDLES:
  # lsof (if available):
  lsof +D /var/log/                          # all open files in dir
  lsof /var/log/mysql-slow.log               # specific file

  # /proc method (always works):
  for pid in $(ls /proc|grep '^[0-9]'); do
    for fd in /proc/$pid/fd/*; do
      target=$(readlink $fd 2>/dev/null)
      [[ "$target" == /var/* ]] && echo "$pid $target"
    done
  done

FIND DELETED BUT HELD OPEN (phantom files):
  find /proc/*/fd -ls 2>/dev/null | grep "(deleted)"
  lsof | grep "(deleted)"           # if lsof available

TRUNCATE SAFELY (process keeps writing):
  > /var/log/bigfile.log
  truncate -s 0 /var/log/bigfile.log

CHECK LOGROTATE:
  grep -rl "mysql-slow.log" /etc/logrotate.d/  # is it configured?
  logrotate -d /etc/logrotate.d/nginx          # dry run
  logrotate -vf /etc/logrotate.d/nginx         # force now, verbose
  cat /var/lib/logrotate/status               # last rotation times

QUICK SPACE RECOVERY:
  journalctl --vacuum-size=500M    # trim journal
  apt-get clean                    # clear apt cache  
  docker system prune              # docker garbage
  find /var/log -name "*.gz" -mtime +30 -delete  # old compressed logs

SORT -RH EXPLAINED:
  -r = reverse (highest first)
  -h = human-readable (understands G > M > K)
  Without -h: 891M sorts above 2.3G (wrong!)
  With -rh:   2.3G sorts above 891M (correct!)
```

> **Google interview tip:** They'll push you to explain WHY each step matters, not just the commands. The "deleted but held open" phantom file concept is a classic gotcha they love — know it cold. Also mention monitoring: "I'd set up an alert at 80% so we're never firefighting at 92% again" — that proactive SRE mindset is exactly what Google wants to hear.

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Real-Time Log Timestamping
> **Company:** Adobe | **Difficulty:** Medium
---
#### **Scenario**
You're troubleshooting a service that produces untagged log output when run manually, making it difficult to analyze timing and sequence of events.
#### **Task**
Create a command that reads from standard input line by line and appends the current timestamp to the end of each line as it's read. Test it interactively by piping output to verify it works, then save the solution to a shell script at `/usr/local/bin/timestamp.sh` and make it executable so it can be used in any pipeline.
#### **Example**
```
# Before (untagged log output)
Application started
Processing request #1234
Database connection established
Request completed
```
```
# After (timestamped in real-time)
Application started - 2025-11-06 15:30:45
Processing request #1234 - 2025-11-06 15:30:46
Database connection established - 2025-11-06 15:30:47
Request completed - 2025-11-06 15:30:48
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/real-time-log-timestamping)

## Real-Time Log Timestamping — Full Deep Dive

---

## Part 1: Understand It Simply

### What Are We Actually Solving?

```
Service runs and outputs logs with NO timestamps:
  Application started
  Processing request #1234         ← WHEN did this happen?
  Database connection established  ← HOW LONG after the previous line?
  Request completed                ← impossible to know

We need a WRAPPER that adds time to each line AS IT ARRIVES:
  command | timestamp.sh
  Application started              - 2025-11-06 15:30:45
  Processing request #1234         - 2025-11-06 15:30:46  ← 1 second later
  Database connection established  - 2025-11-06 15:30:47  ← 1 second later
  Request completed                - 2025-11-06 15:30:48  ← 1 second later
```

### The Core Challenge — What "Real-Time" Means

```
Wrong approach (one timestamp for ALL lines):
  START_TIME=$(date)              ← captured ONCE at start
  cat | while read line; do
    echo "$line - $START_TIME"   ← every line gets same time ✗
  done

Right approach (fresh timestamp PER LINE):
  while read line; do
    echo "$line - $(date)"       ← date called EVERY iteration ✓
  done
  Each line gets the time IT arrives — true real-time ✓
```

### Three Methods — Ranked Best to Worst

```
Method 1: while read loop    ← most readable, truly per-line, portable
Method 2: awk strftime       ← fastest (no subshell), good for high volume
Method 3: ts (moreutils)     ← best tool but requires installation
```

---

## Part 2: Build and Execute

### Step 1 — Understand `while IFS= read -r line`### Step 2 — Prove It's Truly Per-Line Real-Time`while read` correctly shows `:16`, `:17`, `:18` — one second each. `awk` batches them. **`while read` wins for real-time streaming.**

### Step 3 — Create the Script### Step 4 — Test All Scenarios`First :00` → `Second :01` → `Third :02`. Script works exactly as required.

---

## Part 3: Every Concept Explained Simply

### `while IFS= read -r line` — Every Component

```
while                    ← loop until condition fails
  IFS=                   ← Internal Field Separator = EMPTY
                            empty means: don't split on any whitespace
                            result: leading/trailing spaces PRESERVED
                            without: "  hello  " → "hello" (stripped)

  read                   ← bash builtin: reads one line from stdin
                           stops at newline or EOF
                           sets exit code 1 at EOF → while loop ends

       -r                ← raw mode: don't interpret backslashes
                            without: \n = newline, \t = tab (destructive!)
                            with:    \n stays as literal \n ✓

          line           ← variable name to store the line read
; do
    echo "${line}..."    ← process the line
done                     ← back to while → reads next line
```

### Why `$(date)` in a Loop is Correct Here

```
WRONG — timestamp captured once:
  TS=$(date '+%Y-%m-%d %H:%M:%S')  ← evaluated ONCE before loop
  while IFS= read -r line; do
      echo "${line} - ${TS}"        ← all lines get same time
  done

CORRECT — timestamp evaluated per iteration:
  while IFS= read -r line; do
      echo "${line} - $(date '+%Y-%m-%d %H:%M:%S')"  ← fresh per line ✓
  done

$() in a loop = subshell spawned for EACH iteration
Performance: ~1ms per call, fine for log rates up to ~1000 lines/sec
For higher rates: use awk strftime (no subshell) with stdbuf -oL
```

### The Buffering Problem — Pipes and `stdbuf`

```
Pipeline:  command → pipe → timestamp.sh

Two levels of buffering:
  1. command's stdout buffer  → typically line-buffered in terminal
                              → block-buffered (4KB) in pipe ← ISSUE
  2. timestamp.sh's processing → line-by-line ✓

Problem scenario:
  Long-running service writes 10 lines/sec
  Without unbuffering: lines accumulate in 4KB buffer
  Appear in timestamp.sh in burst of ~50 lines at once
  All with the same timestamp ← defeats real-time purpose

Fix: stdbuf -oL forces line-buffering on command's stdout:
  stdbuf -oL ./service.sh | timestamp.sh  ← truly per-line ✓

Or: force unbuffered with -u0:
  stdbuf -oL -eL ./service.sh | timestamp.sh
```

### The `date` Format Codes

```
%Y  → 4-digit year        (2026)
%m  → 2-digit month       (05)
%d  → 2-digit day         (21)
%H  → hour 24h format     (18)
%M  → minute              (16)
%S  → second              (35)
%N  → nanoseconds         (923847261)  ← for milliseconds: %.3N

Common formats:
  '%Y-%m-%d %H:%M:%S'     → 2026-05-21 18:16:35  (task format)
  '%H:%M:%S'              → 18:16:35              (time only)
  '%Y-%m-%dT%H:%M:%S%z'   → 2026-05-21T18:16:35+0000  (ISO 8601)
  '%s'                    → 1747851395             (Unix epoch)
  '%Y-%m-%d %H:%M:%S.%3N' → 2026-05-21 18:16:35.142  (milliseconds)
```

### The Final-Line Edge Case

```
Normal lines end with \n:
  "Application started\n"  ← read consumes \n, stores "Application started"

Final line WITHOUT \n (common in piped output):
  "Request completed"  ← no newline at end
  read: stores "Request completed", returns exit code 1 (EOF)
  Loop exits BUT $line is NOT empty — still has the last line!

Without edge case handling:
  printf "line1\nline2" | while IFS= read -r line; do echo "$line - ts"; done
  → "line1 - ts" only!  ← line2 silently dropped ✗

With edge case handling:
  done
  [ -n "$line" ] && echo "${line} - $(date ...)"
  → "line1 - ts"
  → "line2 - ts" ← properly handled ✓
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "Explain `while IFS= read -r line`. What does each part do?"

**Answer:**
> *"`while` loops continuously reading from stdin until `read` returns non-zero (EOF). `IFS=` sets the Internal Field Separator to empty string — without this, `read` strips leading and trailing whitespace, which would corrupt log lines with intentional indentation. `-r` disables backslash interpretation — without it, `\n` in a log line becomes an actual newline, `\t` becomes a tab, destroying the original content. Together, `IFS= read -r` guarantees the line variable contains exactly what was written to stdin."*

---

### Q2. "Why is `$(date)` called inside the loop rather than before it?"

**Answer:**
> *"If you capture `TS=$(date)` before the loop, every line gets the same timestamp — the moment the script started. That defeats the entire purpose. By calling `$(date)` inside the loop, a fresh subprocess is spawned for every line read. Each line gets the actual wall-clock time at the moment that specific line arrived from stdin. It's slightly less efficient — one subshell per line — but for any reasonable log rate it's imperceptible, and for very high-throughput scenarios you'd switch to awk's built-in `strftime()` which has no subshell overhead."*

---

### Q3. "What is pipe buffering and how does it affect real-time timestamping?"

**Answer:**
> *"When a process writes to a pipe, the OS buffers that output — typically 4KB or 8KB blocks. The receiving process (timestamp.sh) doesn't see anything until the buffer fills or the sender flushes. This means a service writing slowly could produce 50 lines over 5 seconds, but timestamp.sh only sees them all at once when the buffer fills — all getting the same burst timestamp. The fix is `stdbuf -oL` on the upstream command, which forces line-buffered output so each newline flushes the buffer immediately. Alternatively, many programs accept `-u` or `--unbuffered` flags for the same effect."*

```bash
# Default (blocks):    lines batch up → all same timestamp
./service.sh | timestamp.sh

# Fix (line-buffered): each line flushed immediately
stdbuf -oL ./service.sh | timestamp.sh
```

---

### Q4. "Why `#!/bin/bash` and not `#!/bin/sh`?"

**Answer:**
> *"`/bin/sh` is the POSIX shell — it may be `dash`, `ash`, or another minimal shell that doesn't support all bash features. Our script uses `${var:-default}` for optional env variables which is POSIX-compatible, but the general rule is: if you write `bash` features, declare bash. Also, `while IFS= read -r` behavior can vary slightly between shell implementations — bash's behavior is consistent and well-documented. At Adobe's scale where scripts run across many different systems, being explicit about the interpreter prevents subtle breakage from different `sh` implementations."*

---

### Q5. "How would you add millisecond precision to the timestamps?"

**Answer:**
```bash
# GNU date — milliseconds with %3N (first 3 digits of nanoseconds)
echo "event" | while IFS= read -r line; do
    echo "${line} - $(date '+%Y-%m-%d %H:%M:%S.%3N')"
done
# Output: event - 2026-05-21 18:16:35.142

# awk with gensub for ms — requires gawk
echo "event" | awk '{
    cmd = "date +%s%3N"
    cmd | getline ms
    close(cmd)
    printf "%s - %s.%s\n", $0, substr(ms,1,10), substr(ms,11,3)
    fflush()
}'

# Python (most accurate):
echo "event" | python3 -c "
import sys, datetime
for line in sys.stdin:
    ts = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S.%f')[:-3]
    print(line.rstrip() + ' - ' + ts)
    sys.stdout.flush()
"
```

---

### Q6. "How would you make `timestamp.sh` support both appending and prepending the timestamp?"

**Answer:**
```bash
#!/bin/bash
# Enhanced with mode selection
MODE="${TIMESTAMP_MODE:-append}"    # "append" or "prepend"
FORMAT="${TIMESTAMP_FORMAT:-%Y-%m-%d %H:%M:%S}"
SEP="${SEPARATOR:- - }"

while IFS= read -r line; do
    TS=$(date "+$FORMAT")
    case "$MODE" in
        prepend)  echo "${TS}${SEP}${line}" ;;
        append)   echo "${line}${SEP}${TS}"  ;;
        *)        echo "${line}${SEP}${TS}"  ;;
    esac
done
[ -n "$line" ] && {
    TS=$(date "+$FORMAT")
    case "$MODE" in
        prepend) echo "${TS}${SEP}${line}" ;;
        *)       echo "${line}${SEP}${TS}" ;;
    esac
}
```

```bash
# Usage:
./service.sh | timestamp.sh                         # append (default)
TIMESTAMP_MODE=prepend ./service.sh | timestamp.sh  # prepend
```

---

### Q7. "What's the difference between `echo` and `printf` for output in this script?"

**Answer:**
> *"`echo` appends a newline automatically and is simpler. But it has a portability issue: `echo -e` expands escape sequences in some shells but not others, and `echo` with strings starting with `-` can be misinterpreted as flags. `printf` is fully portable and predictable — `printf '%s\n' "$line - $ts"` always works correctly regardless of the content of `$line`. For this script, `echo` is fine because we control what we're outputting and we want the newline. But in a hardened production script, `printf '%s\n' "${line}${SEPARATOR}${TS}"` is the safer choice."*

---

### Q8. "How would you use `timestamp.sh` in a systemd service to add timestamps to all output?"

**Answer:**
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application

[Service]
Type=simple
# Pipe all stdout through timestamp.sh before journald
ExecStart=/bin/bash -c '/usr/local/bin/myapp 2>&1 | /usr/local/bin/timestamp.sh'
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

> *"But actually, for systemd services the better answer is that journald already adds timestamps automatically — every journal entry has a precise timestamp. `timestamp.sh` is most valuable for: ad-hoc debugging sessions where you pipe a service's output to a file, legacy services that write to flat files without systemd, CI/CD pipelines where you want timestamped build logs, or any context where you're capturing stdout to a log file without a logging framework."*

---

## Part 5: Cheat Sheet

```
THE CORE SCRIPT:
  while IFS= read -r line; do
      echo "${line} - $(date '+%Y-%m-%d %H:%M:%S')"
  done
  [ -n "$line" ] && echo "${line} - $(date '+%Y-%m-%d %H:%M:%S')"

KEY FLAGS:
  IFS=      → don't strip whitespace from line edges
  read -r   → don't interpret \n \t as special chars
  done → [ -n "$line" ] → handle final line without newline

DATE FORMAT CODES:
  %Y-%m-%d  → 2026-05-21
  %H:%M:%S  → 18:16:35
  %3N       → 142 (milliseconds, GNU date only)
  %s        → 1747851395 (Unix epoch)
  %z        → +0000 (timezone offset)

USAGE PATTERNS:
  command | timestamp.sh                          # basic
  command | timestamp.sh | tee service.log        # display + save
  command | timestamp.sh >> /var/log/out.log      # append to log
  stdbuf -oL command | timestamp.sh              # force line buffering
  TIMESTAMP_FORMAT="%H:%M:%S" command | timestamp.sh  # custom format

AWK ALTERNATIVE (faster for high volume):
  command | stdbuf -oL awk '{
      print $0 " - " strftime("%Y-%m-%d %H:%M:%S"); fflush()
  }'

WHILE READ vs AWK:
  while read → truly per-line, no buffering surprise, readable
  awk strftime → faster (no subshell), but needs stdbuf -oL in pipes
  ts (moreutils) → best tool, but requires apt install moreutils

PERMISSION CHECK:
  chmod +x /usr/local/bin/timestamp.sh   # make executable
  ls -lh /usr/local/bin/timestamp.sh    # verify -rwxr-xr-x
```

> **Adobe interview tip:** They care about **creative pipeline thinking** and **developer tools**. Show you understand the full context — when does a service NOT have timestamps (docker logs, debug sessions, CI pipelines) and how `timestamp.sh` fits into a broader observability strategy. Mention `ts` from moreutils as the production-grade tool, but know the `while read` implementation cold — that's what they're testing here. Extra points for mentioning the buffering problem and `stdbuf -oL`.

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++


# Recursive Database File Backup
> **Company:** GitLab | **Difficulty:** Easy
---
#### **Scenario**
Before performing a system-wide database schema migration, you've been asked to ensure that all existing `.db` files are safely backed up. These files may be scattered across multiple subdirectories, and simply renaming them isn't enough. You must create backup copies with the `.db.bak` extension, preserving directory structure and permissions.
#### **Task**
Recursively scan a given directory (`/opt/data/`) and create backup copies of all files ending with `.db`. Each backup should have the same name but with the `.bak` suffix added (e.g., `app.db` → `app.db.bak`) and remain in the same directory as the original file, ensuring the originals are left untouched.
#### **Example**
```
# Before (only original .db files exist)
/opt/data/apps/inventory.db
/opt/data/logs/session.db
/opt/data/config/settings.db
```
```
# After (backup copies created alongside originals)
/opt/data/apps/inventory.db
/opt/data/apps/inventory.db.bak
/opt/data/logs/session.db
/opt/data/logs/session.db.bak
/opt/data/config/settings.db
/opt/data/config/settings.db.bak
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/recursive-database-file-backup)


## Recursive Database File Backup — Full Deep Dive

---

## Part 1: Understand It Simply

### What Are We Doing and Why?

```
Before schema migration (dangerous operation):
  /opt/data/apps/inventory.db    ← original (at risk)
  /opt/data/logs/session.db      ← original (at risk)
  /opt/data/config/settings.db   ← original (at risk)

After backup (safe to migrate):
  /opt/data/apps/inventory.db        ← original (untouched)
  /opt/data/apps/inventory.db.bak    ← backup copy (safe)
  /opt/data/logs/session.db          ← original (untouched)
  /opt/data/logs/session.db.bak      ← backup copy (safe)
  /opt/data/config/settings.db       ← original (untouched)
  /opt/data/config/settings.db.bak   ← backup copy (safe)

Rules:
  ✓ Same directory as original
  ✓ Same permissions as original  (cp -p preserves this)
  ✓ Original untouched            (cp, not mv)
  ✓ Recursive                     (find scans all subdirs)
  ✗ Don't backup .bak files       (avoid double-backing up)
```

### The Core Tool: `cp` vs `mv` vs `ln`

```
cp original original.bak   ← COPY — two files, both intact ✓
mv original original.bak   ← MOVE — renames, original GONE ✗
ln original original.bak   ← HARDLINK — shared data, risky ✗

We need cp — always creates an independent copy
```

---

## Part 2: Build and Execute

### Step 1 — Create the Test Environment### Step 2 — Understand the Pipeline Stage by Stage### Step 3 — Three Methods (All Correct, Best to Know All)### Step 4 — Verify EverythingAll 5 `.db` files backed up. Permissions identical. Originals untouched. Non-`.db` files ignored.

---

### Step 5 — Production-Grade Script Version---

## Part 3: Every Concept Explained Simply

### `find` Flags — What Each One Does

```bash
find /opt/data -name "*.db" -type f

find          → the command
/opt/data     → WHERE to start searching (recursive by default)
-name "*.db"  → match filenames ending in .db
                * = any characters
                .db must be at the END of filename
                  inventory.db  ✓
                  session.db    ✓
                  settings.conf ✗ (doesn't end in .db)
                  app.db.bak    ✗ (doesn't end in .db)  ← important!

-type f       → only regular files (not directories)
                if someone made a dir called "data.db/"
                -type f excludes it — we only backup actual files
```

### `cp -p` — Preserve Flag Breakdown

```bash
cp -p source destination

-p preserves:
  Mode        (permissions: -rw-r-----)
  Ownership   (user:group)
  Timestamps  (mtime, atime)

Without -p:
  inventory.db  →  permissions: -rw-r-----  (640)
  inventory.db.bak → permissions: -rw-r--r-- (644) ← umask applied, DIFFERENT

With -p:
  inventory.db  →  permissions: -rw-r-----  (640)
  inventory.db.bak → permissions: -rw-r----- (640) ← IDENTICAL ✓

Why permissions matter for database backups:
  DB file: -rw------- (600) only owner can read
  Without -p: backup is -rw-r--r-- (644) — world readable!
  Potential security issue — sensitive data exposed
```

### `"${f}.bak"` — Variable Expansion and Why Quotes Matter

```bash
f="/opt/data/apps/inventory.db"

"${f}.bak"
  ${f}  → /opt/data/apps/inventory.db
  .bak  → literal string appended
  =     → /opt/data/apps/inventory.db.bak  ✓

Why double quotes?
  f="/opt/data/my databases/app.db"  ← space in path!
  Without quotes: cp -p /opt/data/my databases/app.db ...
                  bash splits on space → BROKEN
  With quotes:    cp -p "/opt/data/my databases/app.db" "..."
                  path treated as one argument → WORKS ✓

Rule: ALWAYS quote variables in shell scripts.
```

### `! -name "*.db.bak"` — Preventing Re-backup

```
If you run the script TWICE without this filter:

First run:
  inventory.db → inventory.db.bak         ✓

Second run (without filter):
  inventory.db → inventory.db.bak         (overwrites, fine)
  inventory.db.bak → inventory.db.bak.bak ← UNWANTED!
  
Third run:
  inventory.db.bak.bak → inventory.db.bak.bak.bak ← cascade!

Fix: exclude .db.bak from find results
  find /opt/data -type f -name "*.db" ! -name "*.db.bak"
  Now: inventory.db.bak doesn't match *.db pattern at all ✓
  (*.db matches files ending in .db — .db.bak ends in .bak)
```

### `< <(find ...)` — Process Substitution vs Pipe

```bash
# Pipe version (common but has a subtle issue):
find /opt/data -name "*.db" | while read f; do
    SUCCESS=$((SUCCESS + 1))  # ← runs in SUBSHELL
done
echo "$SUCCESS"  # ← prints 0! variable not visible outside pipe

# Process substitution (correct):
while read f; do
    SUCCESS=$((SUCCESS + 1))  # ← runs in CURRENT shell
done < <(find /opt/data -name "*.db")
echo "$SUCCESS"  # ← prints 5 ✓

Why: pipe creates a subshell for each side
     < <(...) redirects find output INTO current shell's while loop
     Variables set in the loop are visible after it ✓
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "Walk me through the command to recursively backup all .db files."

**Answer:**
```bash
find /opt/data -type f -name "*.db" ! -name "*.db.bak" | \
    while IFS= read -r f; do
        cp -p "$f" "${f}.bak"
    done
```

> *"`find` recursively walks `/opt/data` and outputs every regular file ending in `.db`. `! -name '*.db.bak'` prevents already-backed-up files from being re-processed. The `while IFS= read -r` loop reads each path safely — `IFS=` preserves spaces in paths, `-r` prevents backslash interpretation. `cp -p` creates the backup in the same directory by appending `.bak` to the full path, with `-p` preserving the original file's permissions and timestamps."*

---

### Q2. "Why use `cp -p` instead of just `cp`?"

**Answer:**
> *"`-p` preserves three things: file mode (permissions), ownership, and timestamps. For database files specifically, permissions often restrict access — a file with `600` is owner-only readable. Without `-p`, the backup would inherit the umask permissions (typically `644`), making sensitive database contents world-readable. Preserving timestamps also matters for auditing — you can verify the backup was made from the pre-migration file, not a post-migration copy. In a GitLab context where you're handling potentially sensitive data, `-p` is non-negotiable."*

---

### Q3. "What's the difference between `-name '*.db'` and `-name '*.db.bak'` matching? Would `*.db` match `app.db.bak`?"

**Answer:**
> *"No — `*.db` matches files whose name ends exactly in `.db`. `app.db.bak` ends in `.bak`, not `.db`, so it doesn't match. This is actually a useful safety property — it means we don't need an explicit exclusion of `.db.bak` files from a pure `find` perspective. However, I add `! -name '*.db.bak'` anyway as defensive programming — it makes the intent explicit and protects against edge cases where someone might create a file named `app.db.bak.db`."*

---

### Q4. "Why `IFS= read -r` instead of just `read`?"

**Answer:**
> *"Two protections. `IFS=` prevents `read` from stripping leading and trailing whitespace from the line — relevant if a database file is in a path like `/opt/data/my app/data.db` where path components could have spaces. `-r` disables backslash interpretation — without it, a filename like `backup\data.db` would have `\d` interpreted as a literal `d`, corrupting the path. Together they guarantee the variable receives exactly the path that `find` output, character for character."*

---

### Q5. "What's the risk of using a pipe to while vs process substitution? When does it matter?"

**Answer:**
> *"Piping to `while` runs the loop body in a subshell in bash — any variables set inside the loop, like counters for success/failure, are invisible to the parent shell after the loop ends. Process substitution `< <(find ...)` redirects find's output into the current shell's while loop, so counters and state changes persist. For a simple backup one-liner this doesn't matter. For a production script that counts successes and failures and exits with the right code — it matters a lot."*

```bash
# Pipe — counter lost:
count=0
find . -name "*.db" | while read f; do count=$((count+1)); done
echo $count  # 0 — lost in subshell

# Process substitution — counter preserved:
count=0
while read f; do count=$((count+1)); done < <(find . -name "*.db")
echo $count  # 5 ✓
```

---

### Q6. "How would you verify the backups are identical to the originals?"

**Answer:**
```bash
# MD5 checksum comparison
find /opt/data -name "*.db" ! -name "*.db.bak" | while IFS= read -r f; do
    orig=$(md5sum "$f" | awk '{print $1}')
    bak=$(md5sum "${f}.bak" 2>/dev/null | awk '{print $1}')
    if [ "$orig" = "$bak" ]; then
        echo "✓ OK: $f"
    else
        echo "✗ MISMATCH or MISSING: $f"
    fi
done

# Alternative: diff (for text files)
diff "$f" "${f}.bak" && echo "identical" || echo "different"

# For binary files: cmp is faster than diff
cmp -s "$f" "${f}.bak" && echo "identical" || echo "different"
```

---

### Q7. "How would you make the backup idempotent — safe to run multiple times?"

**Answer:**
> *"Two approaches: skip if backup already exists and is newer than the original, or always overwrite (idempotent by definition). For a pre-migration backup, I prefer the 'skip if newer' approach — it protects against accidentally overwriting a backup made after the migration with a corrupted post-migration copy:"*

```bash
find /opt/data -type f -name "*.db" ! -name "*.db.bak" | while IFS= read -r f; do
    BAK="${f}.bak"
    # Skip if backup exists AND is newer than original
    if [ -f "$BAK" ] && [ "$BAK" -nt "$f" ]; then
        echo "SKIP (up-to-date): $BAK"
        continue
    fi
    cp -p "$f" "$BAK" && echo "OK: $f" || echo "FAIL: $f" >&2
done
```

---

### Q8. "How would you restore from backup if the migration goes wrong?"

**Answer:**
```bash
# Restore all .bak files to original names
find /opt/data -name "*.db.bak" -type f | while IFS= read -r bak; do
    original="${bak%.bak}"    # strip .bak suffix
    echo "Restoring: $bak → $original"
    cp -p "$bak" "$original"
done

# Or with mv (faster, no copy — destructive if restore fails midway):
find /opt/data -name "*.db.bak" -type f -exec sh -c \
    'mv "$1" "${1%.bak}"' _ {} \;

# Verify restore:
find /opt/data -name "*.db" ! -name "*.db.bak" | while IFS= read -r f; do
    md5sum "$f"
done
```

---

## Part 5: Cheat Sheet

```
CORE ONE-LINER:
  find /opt/data -type f -name "*.db" ! -name "*.db.bak" | \
      while IFS= read -r f; do cp -p "$f" "${f}.bak"; done

THREE METHODS:
  # 1. while read (most readable)
  find /opt/data -type f -name "*.db" | while IFS= read -r f; do
      cp -p "$f" "${f}.bak"
  done

  # 2. find -exec (no subshell)
  find /opt/data -type f -name "*.db" \
      -exec sh -c 'cp -p "$1" "$1.bak"' _ {} \;

  # 3. xargs (parallelizable with -P)
  find /opt/data -type f -name "*.db" -print0 | \
      xargs -0 -I{} cp -p {} {}.bak

KEY FLAGS:
  find -type f      → files only (not directories)
  find -name "*.db" → exact suffix match
  find ! -name "*.db.bak" → exclude already-backed-up files
  cp -p             → preserve permissions, ownership, timestamps
  IFS= read -r      → safe path reading (spaces + backslashes)
  "${f}.bak"        → append .bak — keeps file in same dir ✓

VERIFY BACKUP:
  md5sum "$f" "${f}.bak"          # checksum compare
  cmp -s "$f" "${f}.bak"          # binary compare (silent)
  diff "$f" "${f}.bak"            # diff (text files)

RESTORE:
  find /opt/data -name "*.db.bak" | while IFS= read -r b; do
      cp -p "$b" "${b%.bak}"
  done
  # ${b%.bak} strips the .bak suffix → original filename

DOES *.db MATCH app.db.bak?
  NO — *.db matches files ending in .db
  app.db.bak ends in .bak → excluded automatically ✓
```

> **GitLab interview tip:** They care deeply about **idempotency** (safe to run multiple times) and **data integrity** — always mention MD5 verification after backup and the restore procedure. GitLab runs CI/CD pipelines where scripts may be re-triggered, so showing you think about "what if this runs twice?" puts you ahead. Mentioning `cp -p` for permission preservation specifically — not just `cp` — shows production awareness they value highly.

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++



# Recursive Keyword Finder
> **Company:** X | **Difficulty:** Easy
---
#### **Scenario**
Multiple applications write logs under `/var/log`, and you need to quickly check if any recent errors have been recorded.
#### **Task**
Search recursively under `/var/log` for all files ending with `.log`, print every line that contains the text `ERROR`, ensure the output includes both the filename and the matching line, and save the results to `/home/devops/error_logs.txt`.
#### **Example**
```
# Before (multiple log files scattered across directories)
/var/log/apache2/error.log
/var/log/apache2/access.log
/var/log/application.log
/var/log/mysql/error.log
/var/log/syslog
```
```
# After (ERROR lines found and saved to /home/devops/error_logs.txt)
/var/log/apache2/error.log:ERROR: Connection timeout to database server
/var/log/application.log:ERROR: Unable to write to cache directory
/var/log/mysql/error.log:ERROR: InnoDB: Cannot allocate memory for buffer pool
/var/log/syslog:ERROR: Disk quota exceeded for user appuser
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/recursive-keyword-finder)



## Recursive Keyword Finder — Full Deep Dive

---

## Part 1: Understand It Simply

### What `grep` Does Under the Hood

```
grep  =  Global Regular Expression Print

Opens each file → reads line by line →
tests each line against pattern →
prints matching lines WITH filename prefix

Without -r:  grep "ERROR" /var/log/app.log
             → searches ONE file only

With -r:     grep -r "ERROR" /var/log/
             → walks entire directory tree, opens EVERY file
             → searches thousands of files with one command
```

### What the Output Looks Like

```
grep -r --include="*.log" "ERROR" /var/log/

Format: FILENAME:MATCHING_LINE
         ↑         ↑
         always    the full line that matched
         included
         when
         multiple
         files

/var/log/apache2/error.log:ERROR: Connection timeout
/var/log/mysql/error.log:ERROR: InnoDB: Cannot allocate memory
/var/log/application.log:ERROR: Unable to write to cache
```

---

## Part 2: Build and Execute

### Step 1 — Create Realistic Log Environment### Step 2 — Explain Each Flag Before Running### Step 3 — Run the Core Command### Step 4 — Save to File and Verify### Step 5 — Show What Changes With Different Flags---

## Part 3: Every Concept Explained Simply

### How `grep -r` Actually Walks Directories

```
grep -r --include="*.log" "ERROR" /var/log/

  Step 1: open /var/log/
  Step 2: for each entry:
    Is it a file AND name matches *.log?
      → open it, search for "ERROR"
    Is it a directory?
      → recurse into it (same process)
    Is it a file but doesn't match *.log?
      → SKIP (--include filter)

  /var/log/
  ├── syslog.log        ← ✓ matches *.log → search
  ├── apache2/
  │   ├── error.log     ← ✓ matches *.log → search
  │   ├── access.log    ← ✓ matches *.log → search
  │   └── error.txt     ← ✗ doesn't match *.log → SKIP
  └── mysql/
      └── error.log     ← ✓ matches *.log → search
```

### `--include` vs `-name` — Key Difference

```bash
# grep --include filters by FILENAME only
grep -r --include="*.log" "ERROR" /var/log/
# *.log matches the filename component only
# error.log ✓     access.log ✓     syslog ✗

# find -name also matches FILENAME only
find /var/log -name "*.log" -type f
# Same matching rule

# Common confusion: --include="*.log" does NOT match
# /var/log/app/data.log.1 → ends in .1, not .log ✓ correctly excluded
# /var/log/syslog        → no .log extension ✓ correctly excluded
```

### The Filename:Line Output Format

```
grep output with multiple files:
  filename:matching_line

/var/log/apache2/error.log:ERROR: Connection timeout
├──────────────────────────┤├──────────────────────┤
     absolute path              matched line content
        (from grep)             (verbatim from file)
```

Why does grep show filename automatically?

```bash
# Searching ONE file — no filename prefix:
grep "ERROR" /var/log/apache2/error.log
→ ERROR: Connection timeout   ← no filename (obvious which file)

# Searching MULTIPLE files — filename prefix added:
grep "ERROR" /var/log/apache2/error.log /var/log/mysql/error.log
→ /var/log/apache2/error.log:ERROR: Connection timeout
  /var/log/mysql/error.log:ERROR: InnoDB: Cannot allocate memory

# Force filename ALWAYS (even for 1 file):
grep -H "ERROR" /var/log/apache2/error.log
→ /var/log/apache2/error.log:ERROR: Connection timeout

# Suppress filename (even for multiple files):
grep -h "ERROR" /var/log/apache2/error.log /var/log/mysql/error.log
→ ERROR: Connection timeout
→ ERROR: InnoDB: Cannot allocate memory
```

### `>` vs `>>` — Writing to Output File

```bash
grep -r --include="*.log" "ERROR" /var/log/ > /home/devops/error_logs.txt
                                             ↑
                                   OVERWRITE — creates fresh file each run
                                   (correct for this task)

grep -r --include="*.log" "ERROR" /var/log/ >> /home/devops/error_logs.txt
                                             ↑↑
                                   APPEND — adds to existing file
                                   (use for incremental log collection)
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "What is the command to search all `.log` files recursively for `ERROR`?"

**Answer:**
```bash
grep -r --include="*.log" "ERROR" /var/log/ > /home/devops/error_logs.txt
```

> *"`-r` makes grep search recursively through all subdirectories. `--include='*.log'` filters to only files ending in `.log` — without it, grep would attempt to search binary files and configs which produces noise and is slower. `'ERROR'` is the pattern — case-sensitive by default. The filename is automatically prepended to every matching line when multiple files are searched, giving the `filename:line` format the task requires. `>` saves results to the file."*

---

### Q2. "What does `--include` do and what happens without it?"

**Answer:**
> *"`--include='*.log'` tells grep to only open files whose name matches the glob pattern — files ending in `.log`. Without it, grep opens every single file under `/var/log` regardless of type — binary files like `btmp`, `faillog`, `wtmp`, compressed `.xz` files, `.conf` files, everything. This produces garbage output from binary files, potentially crashes on very large files, and is significantly slower. The `--include` filter is essential for targeted log searches."*

---

### Q3. "How does grep know to show the filename in the output?"

**Answer:**
> *"grep shows the filename prefix whenever it searches more than one file — it needs to tell you which file each match came from. With `-r`, grep is effectively searching many files simultaneously, so every match gets `filename:` prepended. You can override this with `-H` to always force the filename (even for single files) or `-h` to suppress it entirely. For saving to a report file, the default behavior is exactly right."*

---

### Q4. "How is `grep -r --include='*.log'` different from `find | xargs grep`?"

**Answer:**
> *"Both achieve the same result but grep's native `-r --include` is simpler and safer. The `find | xargs grep` pattern has a subtle bug with filenames containing spaces — `xargs` word-splits the paths, so a file named `my app.log` becomes two arguments: `my` and `app.log`. You need `find -print0 | xargs -0 grep` to handle this safely. `grep -r` handles all filenames correctly natively. The find approach is useful when you need more complex filtering — by modification time, size, owner — that `--include` can't express."*

```bash
# Risky (space in filename breaks it):
find /var/log -name "*.log" | xargs grep "ERROR"

# Safe:
find /var/log -name "*.log" -print0 | xargs -0 grep -H "ERROR"

# Best (simplest, handles everything):
grep -r --include="*.log" "ERROR" /var/log/
```

---

### Q5. "How would you search case-insensitively? When would you use it?"

**Answer:**
> *"Add `-i` flag. Without it, `ERROR` only matches `ERROR` — not `error`, `Error`, or `Error`. In practice different applications use different conventions: Java apps write `ERROR`, Python writes `error`, syslog uses `error`. For incident investigation where you want to catch ALL error indicators regardless of case, `-i` is safer. For this task, the requirement says `ERROR` specifically, so case-sensitive is correct — you only want lines explicitly tagged as ERROR by the application."*

```bash
grep -ri --include="*.log" "ERROR" /var/log/   # catches ERROR, error, Error
grep -r  --include="*.log" "ERROR" /var/log/   # only ERROR (task requirement)
```

---

### Q6. "How would you show lines around each match — context before and after?"

**Answer:**
> *"grep's `-A`, `-B`, and `-C` flags show context lines around each match. This is critical for incident investigation — an `ERROR` line alone rarely tells you why it happened, but the lines before it show the sequence of events leading up to it."*

```bash
grep -r --include="*.log" -A 2 "ERROR" /var/log/   # 2 lines After
grep -r --include="*.log" -B 3 "ERROR" /var/log/   # 3 lines Before
grep -r --include="*.log" -C 2 "ERROR" /var/log/   # 2 lines Context (both)

# Example output with -C 2:
# /var/log/apache2/error.log-INFO: Retrying connection attempt 1/3   ← before
# /var/log/apache2/error.log:ERROR: Max retries exceeded             ← match
# /var/log/apache2/error.log-INFO: Service entering degraded mode    ← after
```

---

### Q7. "How would you count errors per log file and sort by most errors?"

**Answer:**
```bash
# Count errors per file, sorted descending:
grep -rc --include="*.log" "ERROR" /var/log/ \
  | grep -v ":0" \
  | sort -t: -k2 -rn

# Output:
# /var/log/apache2/error.log:2
# /var/log/mysql/error.log:2
# /var/log/app/service/application.log:2
# /var/log/syslog.log:1
```

> *"`-c` outputs `filename:count` for every file, including zeros. `grep -v ':0'` removes files with no matches. `sort -t: -k2 -rn` sorts by the second colon-delimited field (the count) numerically in reverse. This gives you an instant health dashboard — which service is generating the most errors."*

---

### Q8. "How would you make this a live monitoring command — watching for new errors in real time?"

**Answer:**
> *"For real-time monitoring across multiple log files, `tail -f` with multiple files or `multitail` is the approach. For grep-based alerting, wrap in a loop or use `inotifywait`:"*

```bash
# Watch a single log for new ERRORs:
tail -f /var/log/app/service/application.log | grep "ERROR"

# Watch multiple files simultaneously:
tail -f /var/log/apache2/error.log /var/log/mysql/error.log | grep "ERROR"

# Periodic scan — run every 60 seconds:
watch -n 60 "grep -rc --include='*.log' 'ERROR' /var/log/ | grep -v ':0'"

# Alert via inotifywait (event-driven, no polling):
inotifywait -m -r -e modify /var/log/ --include '.*\.log' |
  while read dir event file; do
    NEW_ERRORS=$(grep "ERROR" "${dir}${file}" | tail -5)
    [ -n "$NEW_ERRORS" ] && echo "NEW ERRORS in ${dir}${file}: $NEW_ERRORS"
  done
```

---

## Part 5: Cheat Sheet

```
CORE COMMAND:
  grep -r --include="*.log" "ERROR" /var/log/ > /home/devops/error_logs.txt

KEY FLAGS:
  -r          → recursive (walk all subdirectories)
  --include   → filter by filename pattern
  -i          → case-insensitive (ERROR, error, Error)
  -n          → show line numbers (file:linenum:match)
  -l          → filenames only (no matching lines)
  -h          → suppress filenames (lines only)
  -H          → always show filenames (even 1 file)
  -c          → count matches per file
  -w          → whole word match only
  -v          → invert (lines NOT matching)
  -A N        → N lines After each match
  -B N        → N lines Before each match
  -C N        → N lines Context (before + after)

OUTPUT FORMAT:
  Default (multi-file): /path/to/file.log:matching line content
  -n added:             /path/to/file.log:42:matching line content
  -l only:              /path/to/file.log

SAVE vs APPEND:
  >  = overwrite (fresh file each run)
  >> = append    (add to existing file)

ALTERNATIVES:
  find /var/log -name "*.log" -print0 | xargs -0 grep -H "ERROR"
  find /var/log -name "*.log" -exec grep -H "ERROR" {} +

COUNT ERRORS PER FILE:
  grep -rc --include="*.log" "ERROR" /var/log/ | grep -v ":0" | sort -t: -k2 -rn

WHICH FILES HAVE ERRORS:
  grep -rl --include="*.log" "ERROR" /var/log/

LIVE MONITORING:
  tail -f /var/log/app.log | grep "ERROR"
  watch -n 30 'grep -rc --include="*.log" "ERROR" /var/log/ | grep -v ":0"'
```

> **X (Twitter) interview tip:** At X's scale with millions of log lines per second, they'll ask about performance — mention `ripgrep` (`rg`) as the production-grade grep alternative that's 10–100x faster due to SIMD and automatic parallelism: `rg -t log "ERROR" /var/log/`. Also mention that for structured logs (JSON), tools like `jq` or `grep` + `jq` combination are more reliable than text grep because log field positions are stable. Shows you think beyond the basics.
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Sorted Log Aggregation
> **Company:** Airbnb | **Difficulty:** Easy
---
#### **Scenario**
You have a log file `/tmp/app_combined.log` with entries from multiple servers in mixed order. Each line contains: timestamp, hostname, and action.
#### **Task**
Sort `/tmp/app_combined.log` by timestamp (earliest first), then by hostname for identical timestamps. Save the output to `/tmp/app_sorted.log`.
#### **Example**
```
# Before (mixed order)
2025-12-18 14:23:45 api02 POST /users/create
2025-12-18 14:22:10 cache01 CACHE_MISS key=session_abc
2025-12-18 14:22:55 api01 GET /products/list
2025-12-18 14:22:10 db02 INSERT INTO orders VALUES
2025-12-18 14:24:12 cache01 CACHE_SET key=user_profile
```
```
# After (sorted by timestamp, then hostname)
2025-12-18 14:22:10 cache01 CACHE_MISS key=session_abc
2025-12-18 14:22:10 db02 INSERT INTO orders VALUES
2025-12-18 14:22:55 api01 GET /products/list
2025-12-18 14:23:45 api02 POST /users/create
2025-12-18 14:24:12 cache01 CACHE_SET key=user_profile
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/sorted-log-aggregation)


## Sorted Log Aggregation — Full Deep Dive

---

## Part 1: Understand It Simply

### Why Sort Logs at All?

```
Multiple servers write to a shared log simultaneously.
Each server writes its own events — order is unpredictable:

api02   writes at 14:23:45  ─┐
cache01 writes at 14:22:10  ─┤─→ /tmp/app_combined.log
api01   writes at 14:22:55  ─┤   (arrival order, not time order)
db02    writes at 14:22:10  ─┘

To debug: "what happened at 14:22:10?"
  Unsorted → you have to scan entire file manually
  Sorted   → all 14:22:10 events are together, in server order
```

### The Log Line Structure

```
2025-12-18 14:22:10 cache01 CACHE_MISS key=session_abc
├──────────┤├───────┤├──────┤├────────────────────────┤
  field 1    field 2  field 3  field 4+
  date       time     hostname  action (rest of line)

sort sees fields separated by WHITESPACE (default)
  $1 = 2025-12-18   (date)
  $2 = 14:22:10     (time)
  $3 = cache01      (hostname)
  $4+= rest of line

Sort key: field 1+2 (timestamp), then field 3 (hostname)
```

### The Two-Level Sort Logic

```
Primary key:   timestamp = fields 1 AND 2
               date (2025-12-18) + time (14:22:10)
               together they form the full timestamp

Secondary key: hostname = field 3
               only used when timestamps are IDENTICAL
               tie-breaker: cache01 before db02 alphabetically

Result:
  14:22:10 cache01  ← same time, cache01 < db02 alphabetically
  14:22:10 db02
  14:22:55 api01    ← different time
  14:23:45 api02
  14:24:12 cache01
```

---

## Part 2: Build and Execute

### Step 1 — Create the Input File### Step 2 — Understand the `sort` Command Stage by Stage### Step 3 — Run the Final Command### Step 4 — Before vs After Side by Side### Step 5 — Common Variations---

## Part 3: Every Concept Explained Simply

### `sort -k` — The Key Specification Fully Decoded

```
sort -k START[,END][OPTIONS]

  START  = which field to begin the key
  END    = which field to end the key (inclusive)
           if omitted: key goes to end of line
  OPTIONS= modifiers for this specific key

-k1,2   → key from field 1 TO field 2 (inclusive)
           "2025-12-18 14:22:10" — both fields together as one key
           ↑ field 1             ↑ field 2

-k3,3   → key from field 3 TO field 3 (just field 3)
           "cache01" — hostname only

Without END (dangerous):
  -k1    → key from field 1 TO END OF LINE
           "2025-12-18 14:22:10 cache01 CACHE_MISS key=session_abc"
           ← entire rest of line! Not what we want
```

### Why `-k1,2` Not `-k1,1 -k2,2`?

```
sort -k1,1 -k2,2   ← two separate keys
  Primary:   field 1 = "2025-12-18"
  Secondary: field 2 = "14:22:10" ← only used when dates are equal
  Result: dates sort first, then times within same date ✓

sort -k1,2         ← one combined key
  Primary: fields 1+2 = "2025-12-18 14:22:10"
  Treated as a SINGLE string for comparison
  Result: same sort ✓

Both work identically for this log format.
sort -k1,2 is more concise and conventional.
```

### How `sort` Determines Field Boundaries

```
Default field separator = any whitespace (space or tab)
Consecutive whitespace = ONE separator

"2025-12-18  14:22:10  cache01  CACHE_MISS"
             ↑↑         ↑↑       ↑↑
         2 spaces    2 spaces  2 spaces
         = still just field separators

All these count as the same field positions ✓

Custom separator with -t:
  sort -t'|' -k1,2  ← pipe-separated fields
  sort -t','  -k1,2  ← CSV
  sort -t':'  -k1,2  ← colon-separated
```

### Why ISO Timestamps Sort Correctly Without `-n`

```
Timestamps in YYYY-MM-DD HH:MM:SS format sort correctly
as plain STRINGS — no special numeric flag needed.

Why? Because the most significant component is leftmost:
  2025-12-18 → year first
  14:22:10   → hour first

String comparison:
  "2025-12-18 14:21:30" < "2025-12-18 14:22:10"
  because at position 14: '1' < '2' in ASCII ✓

This would FAIL with US date format:
  "12/18/2025" — month first, not year
  "12/18/2025" < "01/01/2026" ✗ because "1" > "0" for month
  ISO dates eliminate this problem entirely
```

### `sort` vs `sort -n` vs `sort -h`

```
sort          → lexicographic (string comparison)
               "10" < "9" because '1' < '9' ✗ (bad for numbers)
               "2025-12-18" < "2025-12-19" ✓ (good for ISO dates)

sort -n       → numeric comparison
               10 > 9 ✓ (correct for pure numbers)
               Use for: numeric fields, sizes, counts
               NOT for: timestamps, hostnames, strings

sort -h       → human-readable numeric
               "1G" > "512M" > "100K" ✓
               Use for: du output, file sizes with units

For ISO timestamps → default sort (no flag) is correct ✓
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "Explain `sort -k1,2 -k3,3`. What does each part mean?"

**Answer:**
> *"`-k` specifies a sort key. `-k1,2` means 'use fields 1 through 2 as the primary sort key' — that's the date and time together forming the complete timestamp. `-k3,3` means 'use only field 3 as the secondary sort key' — the hostname, used only when two lines have identical timestamps. Fields are whitespace-delimited by default. The comma notation `START,END` is important — `-k1` without a comma would use everything from field 1 to end of line, which isn't what we want."*

---

### Q2. "Why doesn't timestamp sorting need `-n` (numeric sort)?"

**Answer:**
> *"ISO 8601 timestamps — `YYYY-MM-DD HH:MM:SS` — are specifically designed to sort correctly as plain strings. The most significant component (year) is leftmost, so lexicographic comparison naturally gives chronological order. `'2025-12-18 14:21:30' < '2025-12-18 14:22:10'` works correctly because the sort engine compares character by character from the left. This would break with US-format dates (`MM/DD/YYYY`) where December comes before January in string sort. ISO dates are the correct format for any system that needs sortable timestamps."*

---

### Q3. "What's the difference between `-k1,2` and `-k1,1 -k2,2`?"

**Answer:**
> *"They produce the same result for this log format, but they're conceptually different. `-k1,2` treats fields 1 and 2 as a single combined key — sort uses the date+time string `'2025-12-18 14:22:10'` as one atomic key. `-k1,1 -k2,2` uses field 1 as the primary key and field 2 as a secondary key — sort first sorts by date alone, then by time only when dates are equal. For this problem they're equivalent, but `-k1,2` is more concise and conventional for timestamp sorting."*

---

### Q4. "What does `-k3,3` instead of just `-k3` do? Why the comma?"

**Answer:**
> *"Without the comma — `-k3` — sort uses everything from field 3 to the end of the line as the key. That would mean the hostname PLUS the entire action field becomes the hostname key, which sorts correctly only coincidentally. `-k3,3` precisely specifies field 3 only, stopping before field 4. The `START,END` syntax is a fundamental part of how `sort -k` works — always specify both unless you intentionally want everything to end of line. A common bug is `-k1` when `-k1,1` was intended."*

---

### Q5. "How would you sort if the timestamp were in a different format — like epoch seconds?"

**Answer:**
```bash
# Log with epoch: 1734534230 api01 GET /health
sort -k1,1n -k2,2 /tmp/app_combined.log
# -k1,1n → sort field 1 NUMERICALLY (epoch is a number)
# Without -n: "1734534230" < "999999999" ← wrong (string sort)
# With -n:    1734534230 > 999999999    ← correct (numeric sort)

# Log with US dates: "12/18/2025 14:22:10 api01..."
# Problem: can't sort directly — need to transform first
sort -t'/' -k3,3 -k1,2 file.log  # sort by year(3), then month/day(1,2)
# OR: pre-process with awk to convert to ISO first
awk '{print $3"-"$1"-"$2, $0}' file.log | sort -k1,2 | cut -d' ' -f3-
```

---

### Q6. "How would you merge and sort multiple log files from different servers?"

**Answer:**
```bash
# Method 1: cat all files then sort
cat /var/log/api01.log /var/log/api02.log /var/log/db01.log \
    | sort -k1,2 -k3,3 > /tmp/merged_sorted.log

# Method 2: sort each file first, then merge (faster for large files)
# Pre-sort each file individually:
for f in /var/log/*.log; do
    sort -k1,2 -k3,3 "$f" > "${f}.sorted"
done
# Then merge-sort (efficient — files already sorted):
sort -m -k1,2 -k3,3 /var/log/*.sorted > /tmp/merged_sorted.log
# -m = merge only, assumes inputs are already sorted

# Method 3: glob pattern (cleanest):
sort -k1,2 -k3,3 /var/log/app-server-*.log > /tmp/merged_sorted.log
```

---

### Q7. "What's `sort -s` and when does it matter?"

**Answer:**
> *"`-s` enables stable sort — when two lines compare as equal on the sort key, their original relative order is preserved. GNU sort is stable by default in recent versions, but `-s` guarantees it. It matters when you have additional metadata in lines that you're NOT sorting by, and you want to preserve the insertion order for equal-keyed lines. For example, if two events at the exact same millisecond from the same server arrive in a specific causal order, `-s` ensures that order survives the sort."*

---

### Q8. "How would you verify the sort is correct programmatically?"

**Answer:**
```bash
# Check: is every line's timestamp >= previous line's timestamp?
awk '
    NR > 1 {
        curr = $1 " " $2
        prev_ts = prev
        if (curr < prev_ts) {
            print "SORT ERROR at line " NR ": " $0
            print "  Previous line timestamp: " prev_ts
            exit 1
        }
    }
    { prev = $1 " " $2 }
    END { print "Sort order verified: " NR " lines OK" }
' /tmp/app_sorted.log

# Quick check — pipe through sort again, diff should be empty:
sort -k1,2 -k3,3 /tmp/app_sorted.log > /tmp/re_sorted.log
diff /tmp/app_sorted.log /tmp/re_sorted.log \
    && echo "Sort verified: stable ✓" \
    || echo "Sort not stable ✗"
```

---

## Part 5: Cheat Sheet

```
CORE COMMAND:
  sort -k1,2 -k3,3 /tmp/app_combined.log > /tmp/app_sorted.log

KEY SYNTAX:
  sort -k START,END [OPTIONS]
  -k1,2   → fields 1 through 2 (timestamp = date + time)
  -k3,3   → field 3 only       (hostname)
  -k1,1n  → field 1, numeric   (for epoch/number fields)
  -k1,2r  → fields 1-2, reversed (latest first)

SORT FLAGS:
  -r  → reverse (descending)
  -n  → numeric  (for numbers, not dates)
  -h  → human-readable sizes (1G > 512M > 100K)
  -u  → unique (remove duplicate lines)
  -s  → stable (preserve original order for equal keys)
  -m  → merge (assume inputs already sorted — faster)
  -t  → field separator (default: whitespace)

MULTI-FILE PATTERNS:
  cat *.log | sort -k1,2 -k3,3          # merge then sort
  sort -m -k1,2 *.sorted.log            # merge pre-sorted files

TIMESTAMP FORMAT MATTERS:
  ISO 8601 (YYYY-MM-DD HH:MM:SS) → sorts correctly as string ✓
  US format (MM/DD/YYYY)          → doesn't sort correctly ✗
  Epoch (1734534230)              → needs -n flag             ✓ with -n

COMMON -k MISTAKES:
  -k1   → key from field 1 to END OF LINE (usually wrong)
  -k1,1 → key is field 1 ONLY            (usually correct)
  -k1,2 → key is fields 1 AND 2          (timestamp = date+time)

VERIFY SORT:
  sort -c -k1,2 -k3,3 file.log   # -c checks sort order, exits non-zero if unsorted
```

> **Airbnb interview tip:** They deal with multi-region, multi-datacenter log aggregation at massive scale. Mention `sort -m` for merge-sorting pre-sorted files — it's O(n) instead of O(n log n) and critical when combining large sorted shards. Also bring up that for truly large-scale log sorting, they'd use distributed tools like Spark or BigQuery rather than Unix `sort` — showing you know when to use which tool is exactly the senior-level thinking Airbnb looks for.

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Throttle High I/O Process
> **Company:** Ebay | **Difficulty:** Easy
---
#### **Scenario**
A server is experiencing latency spikes and intermittent timeouts. CPU and memory appear normal, suggesting a disk I/O bottleneck from a runaway process.
#### **Task**
Identify the process causing high disk I/O using real-time monitoring, then apply I/O throttling to reduce its disk priority to idle class without terminating it.
#### **Example**
```
# Before (high I/O process)
Total DISK READ:       125.43 M/s | Total DISK WRITE:        89.32 M/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 5432 be/4 postgres   98.45 M/s   67.23 M/s  0.00 %  95.32 % postgres: autovacuum worker
```
```
# After (I/O throttled)
Total DISK READ:        12.34 M/s | Total DISK WRITE:         8.92 M/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 5432 idle postgres    9.12 M/s    6.45 M/s  0.00 %  12.34 % postgres: autovacuum worker
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/throttle-high-io-process)


## Throttle High I/O Process — Full Deep Dive

---

## Part 1: Understand It Simply

### Why I/O Bottlenecks Feel Like CPU Problems

```
Symptom:  latency spikes, timeouts
First guess: CPU overload?

Check CPU → normal 20%
Check RAM  → normal 40%
           ↓
The hidden culprit: DISK I/O

One process hogging 98 MB/s of disk reads
  → Every other process WAITS in disk queue
  → Requests time out waiting for data
  → Server appears "slow" with normal CPU/RAM
```

### The Linux I/O Scheduling Stack

```
Process A needs disk   ──┐
Process B needs disk   ──┤──► [ I/O Scheduler ] ──► Disk Driver ──► Disk
Process C needs disk   ──┘
                            ↑
                    Decides WHO gets disk access
                    and HOW MUCH bandwidth each gets

Without priorities: everyone competes equally
                    → runaway process monopolizes disk

With ionice:        you assign priority classes
                    → critical processes guaranteed access
                    → runaway process only gets leftovers
```

### The Three I/O Priority Classes

```
┌──────────────────────────────────────────────────┐
│  realtime (class 1)  ← ALWAYS first, max priority │
├──────────────────────────────────────────────────┤
│  best-effort (class 2) ← default for all processes│
│    level 0 (highest)                              │
│    level 1                                        │
│    level 2                                        │
│    level 3                                        │
│    level 4  ← default                             │
│    level 5                                        │
│    level 6                                        │
│    level 7 (lowest within class)                  │
├──────────────────────────────────────────────────┤
│  idle (class 3)  ← ONLY when disk is otherwise idle│
└──────────────────────────────────────────────────┘

Setting runaway process to idle:
  "You may use disk ONLY when nobody else needs it"
  Process continues running — just waits for idle disk time
```

---

## Part 2: Build and Execute

### Step 1 — Simulate the I/O-Heavy Process### Step 2 — Diagnose: Find the I/O Offender### Step 3 — Apply the Fix: `ionice` to Idle Class`none: prio 0` → `idle`. Throttled without terminating.

### Step 4 — Protect Critical Processes at High Priority### Step 5 — Full Summary and Production Patterns---

## Part 3: Every Concept Explained Simply

### `/proc/PID/io` — Reading I/O Without iotop

```
cat /proc/522/io

rchar: 2048000        ← bytes read by process (includes cache)
wchar: 63481716736    ← bytes written by process (includes cache)
syscr: 272089         ← number of read() syscalls
syscw: 157167         ← number of write() syscalls
read_bytes: 1011712   ← bytes actually read from DISK (bypassed cache)
write_bytes: 63481716736 ← bytes actually written to DISK
cancelled_write_bytes: 0 ← writes cancelled before hitting disk

read_bytes and write_bytes = actual disk I/O
(iotop shows these as rates — KB/s, MB/s)

Snapshot-based rate calculation:
  read1 = read_bytes at t=0
  read2 = read_bytes at t=1
  rate  = (read2 - read1) KB/s
```

### The CFQ / BFQ Scheduler — How `ionice` Works Internally

```
ionice sets a flag in the kernel for your process.
The I/O scheduler reads this flag when queuing disk requests.

BFQ (Budget Fair Queuing) — modern Linux default:
  Each process gets a "budget" of I/O bandwidth
  Budget size ∝ I/O priority class + level

  idle class:  budget = ZERO
               queued ONLY when all other budgets exhausted
               "runs in the gaps"

  be/4 (default): budget = baseline
  be/0 (highest): budget = 8x baseline

  Process reads faster → gets MORE budget (reward)
  (BFQ learns access patterns and optimizes)

ionice -c 3 effectively tells BFQ:
  "Don't allocate any budget to this process
   unless the disk is truly sitting idle"
```

### `nice` vs `ionice` — Completely Independent

```
nice / renice   → CPU scheduling priority
                  -20 (greedy) to +19 (generous)
                  Controls how much CPU time a process gets

ionice          → I/O scheduling priority
                  idle/best-effort/realtime
                  Controls how much disk bandwidth a process gets

A process can be:
  CPU-greedy  + I/O-idle    (intensive computation, light disk)
  CPU-nice    + I/O-greedy  (idle CPU but hammering disk)
  CPU-nice    + I/O-idle    (fully backgrounded job)  ← our use case

For backup jobs: always set BOTH:
  nice -n 19 ionice -c 3 rsync /data /backup
  ↑ CPU gentle  ↑ I/O gentle
```

### Why `ionice` Doesn't Always Work (NVMe Warning)

```
ionice works with: CFQ, BFQ schedulers (traditional HDDs, some SSDs)

NVMe drives use "none" or "mq-deadline" scheduler:
  cat /sys/block/nvme0n1/queue/scheduler
  [none] mq-deadline

  With "none": I/O scheduling is done by the hardware
               ionice has NO EFFECT

Check your disk's scheduler:
  cat /sys/block/sda/queue/scheduler   # HDD
  cat /sys/block/nvme0n1/queue/scheduler  # NVMe

For NVMe — alternative: cgroups v2 io.weight
  echo "252:0 100" > /sys/fs/cgroup/myapp/io.weight
  (device major:minor + weight 1-10000)
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How do you identify which process is causing disk I/O bottleneck?"

**Answer:**
```bash
# Best tool — iotop (real-time, sorted by I/O):
sudo iotop -bon 2 --only   # batch, 2 samples, active only

# Without iotop — /proc/PID/io (always available):
for pid in /proc/[0-9]*/io; do
    pidnum=$(cut -d/ -f3 <<< "$pid")
    wb=$(awk '/^write_bytes/{print $2}' "$pid" 2>/dev/null || echo 0)
    comm=$(cat "/proc/$pidnum/comm" 2>/dev/null)
    echo "$wb $pidnum $comm"
done | sort -rn | head -5

# iostat for device-level view:
iostat -x 1 3     # 3 samples, 1 second apart, extended stats
```

> *"I start with `iotop -o` to see only active I/O processes in real-time. The `IO>` column shows what percentage of time each process is blocked waiting for disk. Combined with `read_bytes`/`write_bytes` from `/proc/<PID>/io`, I can confirm it's actual disk I/O and not just page cache activity."*

---

### Q2. "What does `ionice -c 3 -p <PID>` do? Explain each flag."

**Answer:**
> *"`ionice` sets the I/O scheduling class and priority for a process. `-c 3` sets class 3 — idle — the lowest possible I/O priority. A process in idle class only gets disk access when no other process needs it — it runs in the gaps. `-p PID` targets an already-running process by its PID rather than launching a new one. The process is NOT killed or paused — it continues executing, but any disk requests it makes are queued behind everyone else's. On a busy server, this can reduce the offending process's I/O from 98 MB/s to under 5 MB/s while keeping critical services unaffected."*

---

### Q3. "What are the three I/O scheduling classes? When would you use each?"

**Answer:**

| Class | Flag | Use Case |
|-------|------|----------|
| `realtime` | `-c 1` | Emergency data recovery, video capture, never for normal services — can starve everything else |
| `best-effort` | `-c 2 -n 0-7` | Normal services; `-n 0` for databases, `-n 4` for apps, `-n 7` for background tasks |
| `idle` | `-c 3` | Backups, rsync, file indexing, anything non-urgent |

> *"For eBay's scenario: the runaway process gets `-c 3` idle. Databases and user-facing services get `-c 2 -n 0`. Most application processes stay at the default `-c 2 -n 4`."*

---

### Q4. "Does `ionice` work on NVMe SSDs? What's the alternative?"

**Answer:**
> *"It depends on the I/O scheduler. `ionice` works with CFQ and BFQ schedulers — historically used for HDDs and some SSDs. Modern NVMe drives typically use the `none` scheduler (no queuing, hardware handles it), making `ionice` have little to no effect. Check with `cat /sys/block/nvme0n1/queue/scheduler`. For NVMe, the correct tool is cgroups v2 `io.weight` — you can set relative I/O bandwidth weights per cgroup:"*

```bash
# Check scheduler:
cat /sys/block/nvme0n1/queue/scheduler

# cgroups v2 weight (works on any scheduler):
echo "252:0 100" > /sys/fs/cgroup/backup_job/io.weight
# Major:minor device number + weight (1-10000, default 100)

# Or via systemd (recommended):
# IOWeight=50 in [Service] section
```

---

### Q5. "How do you make the throttle permanent — surviving process restarts?"

**Answer:**
> *"`ionice` on a PID is ephemeral — when the process exits and restarts, it gets default priority again. For systemd-managed services, set it in the unit file:"*

```ini
# /etc/systemd/system/backup.service
[Service]
IOSchedulingClass=idle       # class 3 — equivalent to ionice -c 3
IOSchedulingPriority=7       # sub-priority (0-7, only used for class 2)
Nice=19                      # CPU throttle too

# For a critical service needing high I/O:
IOSchedulingClass=best-effort
IOSchedulingPriority=0       # highest within class 2
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart backup.service
# Now always starts with idle I/O priority ✓
```

---

### Q6. "What's the difference between `iotop`, `iostat`, and reading `/proc/PID/io`?"

**Answer:**

| Tool | Shows | Use When |
|------|-------|----------|
| `iotop` | Real-time per-process I/O rates (KB/s) | Finding which process is actively hammering disk right now |
| `iostat` | Device-level stats (disk utilization, await time) | Confirming the disk itself is saturated, finding which device |
| `/proc/PID/io` | Cumulative totals since process started | Calculating rates manually, scripting, when iotop unavailable |

> *"`iostat -x` showing `%util` near 100% confirms disk saturation. `iotop -o` identifies the culprit process. `/proc/PID/io` is the fallback when tools aren't installed — it's always available since it's part of the Linux kernel's proc filesystem."*

---

### Q7. "After setting ionice, how do you verify it's working?"

**Answer:**
```bash
# Verify priority was set:
ionice -p <PID>
# Should show: idle

# Verify disk utilization dropped (watch for 10 seconds):
iostat -x 1 10 | grep -E "Device|sda|vda"

# Watch I/O rate for the specific PID (snapshot comparison):
get_io() { awk '/write_bytes/{print $2}' /proc/$1/io 2>/dev/null; }
W1=$(get_io $PID); sleep 2; W2=$(get_io $PID)
echo "Write rate: $(( (W2-W1)/2 )) bytes/sec"

# Or with iotop targeted at just that PID:
iotop -p <PID> -b -n 3
```

---

### Q8. "A backup job is scheduled at 2 AM. How would you ensure it never impacts production I/O?"

**Answer — the complete production approach:**

```bash
# Method 1: wrapper script (immediate)
#!/bin/bash
exec ionice -c 3 nice -n 19 /usr/bin/rsync -av /data /backup

# Method 2: systemd timer + service (best practice)
# /etc/systemd/system/nightly-backup.service
[Unit]
Description=Nightly Backup
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/rsync -av /data /backup
IOSchedulingClass=idle       # never competes with production
IOSchedulingPriority=7
Nice=19
CPUWeight=10                 # also limit CPU (cgroups v2)
MemoryMax=512M              # prevent memory pressure too

# /etc/systemd/system/nightly-backup.timer
[Unit]
Description=Run backup daily at 2 AM

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

> *"Using systemd ensures the I/O class persists across retries, doesn't rely on a wrapper script being called correctly, and integrates with monitoring. The `CPUWeight` and `MemoryMax` settings ensure the backup can't impact production in any resource dimension — not just I/O."*

---

## Part 5: Cheat Sheet

```
FIND I/O OFFENDER:
  iotop -bon 2 --only                    # best tool — real-time rates
  for p in /proc/[0-9]*/io; do           # no tools needed
    echo "$(awk '/write_bytes/{print $2}' $p) $(cut -d/ -f3 <<<$p)"
  done | sort -rn | head -5

THROTTLE (immediate):
  ionice -c 3 -p <PID>                   # set to idle class
  ionice -p <PID>                        # verify: shows "idle"

PROTECT CRITICAL SERVICES:
  ionice -c 2 -n 0 -p $(pgrep postgres)  # highest best-effort
  ionice -c 2 -n 0 -p $(pgrep mysql)

I/O CLASSES:
  -c 1  realtime   ← highest, can starve others
  -c 2  best-effort ← default, use -n 0-7 for sub-priority
  -c 3  idle        ← only when disk idle ← use for throttling

LAUNCH WITH PRIORITY:
  ionice -c 3 rsync /data /backup        # start idle
  nice -n 19 ionice -c 3 rsync ...       # CPU + I/O throttled

PERMANENT (systemd):
  IOSchedulingClass=idle
  IOSchedulingPriority=7   (only for class 2)

CHECK SCHEDULER (ionice only works with bfq/cfq):
  cat /sys/block/sda/queue/scheduler
  [bfq] → ionice works ✓
  [none] → use cgroups io.weight instead

VERIFY DISK PRESSURE:
  iostat -x 1 5 | awk '/^vda|^sda/{print $1, "util:", $NF"%"}'
  cat /proc/<PID>/io | grep bytes
```

> **eBay interview tip:** eBay runs one of the world's highest-volume e-commerce platforms — disk I/O during peak sales events (eBay Open, holiday shopping) is critical. Mention that `ionice` is the immediate fix, but the **production-grade answer** is `IOSchedulingClass=idle` in systemd units for all non-critical batch jobs, combined with disk I/O monitoring alerts in Prometheus/Grafana at 70% `%util` threshold. Prevention beats reaction — that's the senior SRE answer they want.
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Trace Process Service Ownership
> **Company:** NVIDIA | **Difficulty:** Hard
---
#### **Scenario**
A process is consuming excessive resources, but its origin is unclear.
#### **Task**
Create a utility script at `/home/devops/trace_service.sh` that accepts a **PID** argument and outputs its managing systemd service name, full status, and the last 20 log entries.
`trace_service.sh Example:`
```sh
#!/bin/bash
PID=$1
echo "PID: $PID"
# Extract service name managing this PID using systemd
# Hint: systemd can resolve a PID back to its unit
SERVICE=""
echo "SERVICE: $SERVICE"
echo "---- STATUS ----"
# Hint: show full systemd status for the identified service with --no-pager flag
echo "---- LOGS ----"
# Hint: show the last 20 log entries for the service from journald with --no-pager flag
```
#### **Example**
Running the script should produce a structured report similar to this:
```text
root@server:~# ./trace_service.sh 4567
Service Identified: nginx.service
--- Status ---
● nginx.service - A high performance web server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
   Active: active (running) since Fri 2025-12-19 14:00:00 UTC; 1h ago
 Main PID: 4567 (nginx)
--- Last 20 Logs ---
Dec 19 14:00:01 server nginx[4567]: Starting web server...
Dec 19 14:00:02 server nginx[4567]: Configuration loaded successfully.
...
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/trace-process-service-ownership)


## Trace Process Service Ownership — Full Deep Dive

---

## Part 1: Understand It Simply

### The Problem

```
You see this in top/ps:
  PID  %CPU  %MEM  COMMAND
  4567  89.3  45.2  python3

Questions:
  • What systemd service spawned this?
  • Is it supposed to be running like this?
  • What does its full status show?
  • What do its recent logs say?

Without context, you can't answer any of these.
This script automates ALL of them from one PID.
```

### How systemd Tracks Processes — cgroups

```
systemd uses Linux control groups (cgroups) to track processes.
Every process spawned by a service lives in a cgroup named after that service.

Hierarchy:
  /sys/fs/cgroup/
  └── system.slice/
      ├── nginx.service/     ← nginx processes live here
      │   ├── PID 4567
      │   └── PID 4568
      ├── mysql.service/     ← mysql processes live here
      └── sshd.service/

Reading /proc/PID/cgroup reveals:
  0::/system.slice/nginx.service
                   ↑
          the service name is RIGHT THERE
```

### The Two Resolution Paths

```
Path 1: systemctl status <PID>    ← asks systemd directly
         systemd looks up which unit owns this PID
         returns full service status

Path 2: /proc/PID/cgroup           ← reads kernel directly
         parse the cgroup path
         extract service name from path string
         then query systemctl with the service name
```

---

## Part 2: Build and Execute

### Step 1 — Understand the Key Building Blocks### Step 2 — Build the Script Incrementally### Step 3 — Write the Complete Script### Step 4 — Test the Script---

## Part 3: Every Concept Explained Simply

### How systemd Links PIDs to Services — cgroups

```
cgroup = Control Group — Linux kernel mechanism to:
  • Track which processes belong to which group
  • Apply resource limits per group
  • Account for resource usage per group

systemd maps ONE service → ONE cgroup

/sys/fs/cgroup/system.slice/
├── nginx.service/       ← all nginx processes
│   ├── 4566  (master)
│   ├── 4567  (worker)  ← THIS is our mystery PID
│   └── 4568  (worker)
├── mysql.service/
│   └── 3210

Reading /proc/4567/cgroup reveals:
  0::/system.slice/nginx.service
                   ↑
           service name embedded here

This is the KERNEL'S answer — doesn't require
asking systemd at all. Always available.
```

### The Four Service Resolution Methods — Why Each Exists

```
Method 1: /proc/PID/cgroup (cgroup v2)
  When: Modern Linux (kernel 4.5+), systemd 219+
  Path: "0::/system.slice/nginx.service"
  Extract: everything after last slash before .service
  Fastest, no IPC needed

Method 2: systemctl status <PID>
  When: systemd is running as PID 1
  How:  systemd keeps an internal map: PID → unit
        systemctl queries systemd over D-Bus
  Most user-friendly output
  Requires D-Bus connection

Method 3: /proc/PID/cgroup (cgroup v1)
  When: older kernels, hybrid cgroup setups
  Path: "1:name=systemd:/system.slice/nginx.service"
  Same info, different format — needs different parser

Method 4: scan MainPID of all units
  When: methods 1-3 all fail
  Slow: queries every service unit one by one
  Last resort: always works if service is running
```

### `systemctl status <PID>` — Hidden Feature

```bash
# Most people know:
systemctl status nginx.service   ← by service name

# Few people know:
systemctl status 4567             ← by PID number!

systemd does the lookup:
  "Which service contains PID 4567?"
  → checks its internal cgroup map
  → returns full status of nginx.service

Output includes:
  ● nginx.service          ← unit name (extract this)
  Main PID: 4566 (nginx)   ← service's main PID
  CGroup: ...              ← all PIDs in the service
     ├─4566
     ├─4567               ← our original PID is here
     └─4568
```

### `journalctl -u SERVICE -n 20 --no-pager`

```bash
journalctl         → query systemd journal
  -u nginx.service → filter: only this unit's logs
  -n 20            → last 20 lines (like tail -n 20)
  --no-pager       → don't pipe through less
                     output goes directly to stdout ← critical for scripts!

Without --no-pager in a script:
  journalctl would invoke 'less'
  Script hangs waiting for user input
  Output is never captured properly

Other useful journalctl options:
  --since "1 hour ago"   → time-bounded logs
  -p err                 → only error-level entries
  -f                     → follow (like tail -f)
  --output=json          → structured output
  -x                     → add explanatory text
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How does your script map a PID to a systemd service name?"

**Answer:**
> *"I use a four-method cascade in order of reliability. First, I read `/proc/PID/cgroup` and extract the `.service` component from the cgroup path — on modern cgroup v2 systems the path looks like `0::/system.slice/nginx.service`, so the service name is literally embedded in the string. Second, I call `systemctl status <PID>` which asks systemd directly over D-Bus. Third, I handle cgroup v1 format which has a different path structure. Fourth, as a last resort, I scan all units looking for one whose `MainPID` matches. The cascade ensures the script works across different systemd versions and cgroup configurations."*

---

### Q2. "What is a cgroup and why does systemd use them?"

**Answer:**
> *"A cgroup — control group — is a Linux kernel feature for grouping processes together to track and limit their resource usage. systemd creates one cgroup per service unit and puts all processes spawned by that service into it. This solves several problems simultaneously: it tracks all processes belonging to a service (including children and workers), it enforces resource limits defined in the unit file with `MemoryMax`, `CPUWeight` etc., and it enables clean service shutdown by sending signals to the entire cgroup rather than hunting for child processes individually. The cgroup path in `/proc/PID/cgroup` is essentially the kernel saying 'this process is a member of this service's group'."*

---

### Q3. "What does `--no-pager` do and why is it critical in scripts?"

**Answer:**
> *"Without `--no-pager`, both `systemctl` and `journalctl` pipe their output through `less` — an interactive pager that waits for user input. In a script, this causes the script to hang indefinitely waiting for someone to press `q`. `--no-pager` bypasses this and sends output directly to stdout, which the script can capture, pipe, or redirect normally. It's a rule of thumb: any `systemctl` or `journalctl` call in a script must include `--no-pager`. Same principle applies to `git log` with `--no-pager`, `man` with `--no-pager`, etc."*

---

### Q4. "What does `systemctl status <PID>` return that `ps` doesn't?"

**Answer:**
> *"`ps` only knows about the single process with that PID — its CPU, memory, command line. `systemctl status <PID>` shows the full service context: whether the service is enabled to start on boot, how long it's been running, its complete process tree (all worker processes under the same service), recent journal entries, and critically the service name which lets you look up configuration, dependencies, and restart policy. It's the difference between seeing one tree and seeing the entire forest it belongs to."*

---

### Q5. "How would you extend the script to also show resource usage and limits?"

**Answer:**
```bash
# After identifying SERVICE, add resource section:
echo "---- RESOURCE USAGE ----"
systemctl show "$SERVICE" --no-pager \
    -p MemoryCurrent -p CPUUsageNSec -p TasksCurrent \
    -p MemoryMax -p CPUWeight -p TasksMax 2>/dev/null | \
    awk -F= '{
        if($1=="MemoryCurrent") printf "  Memory now:   %.1f MB\n", $2/1048576
        if($1=="MemoryMax")     printf "  Memory limit: %s\n", ($2=="infinity"?"unlimited":$2/1048576" MB")
        if($1=="TasksCurrent")  printf "  Tasks now:    %s\n", $2
        if($1=="CPUWeight")     printf "  CPU weight:   %s\n", $2
    }'
```

---

### Q6. "What if the process was spawned by a script that was spawned by systemd — not a direct child?"

**Answer:**
> *"It still works — cgroups are hierarchical and inherited. When systemd starts a service, all processes it spawns go into that service's cgroup, including grandchildren, great-grandchildren, scripts calling scripts. `/proc/any_descendant_pid/cgroup` shows the ancestor service's cgroup path regardless of depth. This is one of the core values of cgroups over traditional process tracking — you don't need to trace the parent chain manually, the kernel maintains the group membership for you."*

---

### Q7. "How would you add alerting if this script detects a service consuming over a threshold?"

**Answer:**
```bash
# Add to script after extracting SERVICE and resource usage:
MEM_KB=$(awk '/^VmRSS:/{print $2}' /proc/$PID/status 2>/dev/null)
MEM_THRESHOLD=512000  # 512 MB in KB

if [ -n "$MEM_KB" ] && [ "$MEM_KB" -gt "$MEM_THRESHOLD" ]; then
    MSG="ALERT: $SERVICE on $(hostname) is using ${MEM_KB}KB RAM (threshold: ${MEM_THRESHOLD}KB)"
    echo "$MSG"
    # Send alert:
    echo "$MSG" | mail -s "High memory: $SERVICE" ops@nvidia.com
    # Or: curl -X POST https://hooks.slack.com/... -d "{\"text\":\"$MSG\"}"
    # Or: systemctl kill "$SERVICE"  # emergency shutdown
fi
```

---

## Part 5: Cheat Sheet

```
CORE SCRIPT COMMANDS:
  cat /proc/$PID/cgroup                    # raw cgroup path
  grep -oP '(?<=/)[^/]+\.service' ...      # extract service name
  systemctl status $PID --no-pager         # service status by PID
  systemctl status $SERVICE --no-pager     # service status by name
  journalctl -u $SERVICE -n 20 --no-pager  # last 20 log lines

SERVICE RESOLUTION CASCADE:
  1. /proc/$PID/cgroup  (cgroup v2: "0::/system.slice/svc.service")
  2. systemctl status $PID (D-Bus query to systemd)
  3. /proc/$PID/cgroup  (cgroup v1: different format)
  4. scan MainPID of all units (last resort)

KEY /proc FILES:
  /proc/$PID/comm     → process name
  /proc/$PID/cgroup   → cgroup path (contains service name)
  /proc/$PID/status   → UID, PPID, VmRSS, state
  /proc/$PID/cmdline  → full command line (null-separated)
  /proc/$PID/fd/      → open file descriptors

JOURNALCTL FLAGS:
  -u <unit>           → filter by service
  -n 20               → last 20 lines
  --no-pager          → ALWAYS in scripts
  --since "1h ago"    → time range
  -p err              → errors only
  -f                  → follow (like tail -f)
  -x                  → explanatory messages
  --output=json       → structured output

SYSTEMCTL FLAGS FOR SCRIPTS:
  --no-pager          → never use pager (ALWAYS add in scripts)
  show -p MainPID     → get specific property value
  --value             → show only the value, not key=value
  list-units --type=service  → list all services
```

> **NVIDIA interview tip:** They run massive GPU cluster infrastructure where a single runaway process can consume all GPU memory and block other jobs. Mention extending this script with `nvidia-smi` integration — correlate the PID with GPU utilization using `nvidia-smi -q -d PIDS` to show if the process is also a GPU offender. That GPU-aware angle is uniquely relevant at NVIDIA and shows you think beyond generic Linux to their specific domain.

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++


# Tracing Log File Writes
> **Company:** Bloomberg | **Difficulty:** Easy
---
#### **Scenario**
The `/var/log/messages` file has been growing unusually fast, filling up disk space within hours.
#### **Task**
Identify the process that is writing heavily to `/var/log/messages` by monitoring system activity in real time. Save the process details using `ps` and last `50` lines of logs at `/home/devops/excessive_log_process.txt`
#### **Example**
```
# Before (log file growing rapidly)
/var/log/messages: 15 GB and increasing
Disk usage: 92% and climbing
```
```
# After (responsible process identified)
Process identified: rsyslogd (PID 1234)
Confirmed active writes to /var/log/messages
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/tracing-log-file-writes)

## Tracing Log File Writes — Full Deep Dive

---

## Part 1: Understand It Simply

### What Are We Solving?

```
/var/log/messages is growing at GB/hour

Question: WHO is writing to it?

Three categories of writers:

  Category 1: rsyslogd / syslog-ng
    The log daemon itself — receives messages from all processes
    and writes them to /var/log/messages
    THIS is usually the direct file writer

  Category 2: Applications logging via syslog()
    App calls syslog() → kernel → rsyslogd → writes to file
    The app is the SOURCE, rsyslog is the WRITER

  Category 3: Direct file writers
    App opens /var/log/messages directly and writes
    Unusual but possible (e.g., buggy scripts)

We need to find BOTH:
  The direct file writer (has an open fd to the file)
  The source of excessive messages (who is flooding syslog)
```

### The Open File Descriptor Chain

```
Application (PID 1234)
    │
    │ syslog("ERROR: something") ← via syslog() syscall
    │
    ▼
/dev/log socket  ← syslog messages go here
    │
    ▼
rsyslogd (PID 567)
    │
    │ write() ← rsyslogd writes to file
    ▼
/var/log/messages  ← the file growing on disk
    ↑
    fd 7 open in rsyslogd's process
    visible in /proc/567/fd/7 → /var/log/messages
```

---

## Part 2: Build and Execute

### Step 1 — Create the Scenario### Step 2 — Find Processes with the File Open**fd 3 → /var/log/messages** — confirmed open for writing.

### Step 3 — Full Investigation### Step 4 — Save Full Report to FileReport saved with process details + last 50 log lines. Now the deep explanation.

---

## Part 3: Every Concept Explained Simply

### `/proc/PID/fd/` — The Open File Descriptor Map

```
Every process has a directory at /proc/PID/fd/
It contains symbolic links — one per open file descriptor

ls -la /proc/1299/fd/
  lrwxrwxrwx  0 → /dev/null       ← stdin
  lrwxrwxrwx  1 → pipe:[1878]     ← stdout
  lrwxrwxrwx  2 → pipe:[1879]     ← stderr
  lrwxrwxrwx  3 → /var/log/messages ← FOUND IT ← fd 3 is our file!

readlink /proc/1299/fd/3
→ /var/log/messages

This is exactly what lsof reads internally.
No tool installation needed — /proc is always there.
```

### The `/proc/PID/io` File — Measuring Write Volume

```
cat /proc/1299/io

rchar:       37643    ← bytes read by read() syscalls (includes cache)
wchar:         612    ← bytes written by write() syscalls (includes cache)
syscr:          27    ← number of read() system calls
syscw:          12    ← number of write() system calls
read_bytes:      0    ← bytes actually read from PHYSICAL disk
write_bytes:  12288   ← bytes actually written to PHYSICAL disk ← key metric
cancelled_write_bytes: 0

write_bytes is the smoking gun:
  Normal process:       write_bytes = 0-1MB cumulative
  Excessive log writer: write_bytes = growing by GB/hour
  
Rate calculation:
  w1=$(awk '/write_bytes/{print $2}' /proc/PID/io); sleep 5
  w2=$(awk '/write_bytes/{print $2}' /proc/PID/io)
  rate=$(( (w2 - w1) / 5 ))  # bytes per second
```

### `lsof` vs `/proc/fd/` — Two Ways to Same Answer

```
lsof /var/log/messages
  COMMAND  PID  USER  FD  TYPE  SIZE/OFF  NODE  NAME
  python3  1299 root  4w  REG   4096      123   /var/log/messages
  ↑ lsof reads /proc internally and formats it nicely

Manual /proc scan (no tools needed):
  for pid in /proc/[0-9]*/fd; do
    pidnum=$(cut -d/ -f3 <<< "$pid")
    for fd in "$pid"/*; do
      t=$(readlink "$fd" 2>/dev/null)
      [ "$t" = "/var/log/messages" ] && echo "PID $pidnum"
    done
  done

lsof advantages:  formatted output, FD mode (r/w/u), offset
/proc advantages: always available, no installation, scriptable
```

### `inotifywait` — Event-Based File Monitoring

```bash
# Watch for writes to a specific file in real time
inotifywait -m /var/log/messages

Output:
  /var/log/messages MODIFY     ← someone wrote to the file
  /var/log/messages MODIFY
  /var/log/messages ACCESS

# But this only shows THAT writes are happening
# To see WHO is writing — combine with /proc fd scan at the moment:

inotifywait -m /var/log/messages | while read path event file; do
    echo "Write detected!"
    # Immediately scan /proc to find the writer
    for pid in /proc/[0-9]*/fd; do
        pidnum=$(cut -d/ -f3 <<< "$pid")
        readlink ${pid}/* 2>/dev/null | grep -q "$path" && \
            echo "  Writer: PID=$pidnum $(cat /proc/$pidnum/comm)"
    done
done
```

### `ps` Output Fields for Investigation

```bash
ps -p 1299 -o pid,ppid,user,stat,pcpu,pmem,rss,lstart,comm

  PID   = process ID
  PPID  = parent process ID (who spawned it)
  USER  = owner
  STAT  = S=sleeping R=running Z=zombie D=disk wait
  %CPU  = CPU usage
  %MEM  = memory percentage
  RSS   = resident set size (actual RAM in KB)
  LSTART= when it started (full date+time)
  COMM  = command name (no path, no args)

For more detail:
  ps -p 1299 -o pid,ppid,user,stat,pcpu,pmem,rss,lstart,cmd
  CMD = full command with arguments ← shows the actual script/binary
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How do you find which process is writing to a specific log file?"

**Answer:**
```bash
# Primary method — /proc fd scan (no tools needed):
for pid in /proc/[0-9]*/fd; do
    pidnum=$(echo "$pid" | cut -d/ -f3)
    for fd in "$pid"/*; do
        [ "$(readlink "$fd" 2>/dev/null)" = "/var/log/messages" ] && \
            echo "PID $pidnum ($(cat /proc/$pidnum/comm)) has it open"
    done
done

# With lsof (if installed):
lsof /var/log/messages

# Quick one-liner:
grep -rl "/var/log/messages" /proc/[0-9]*/fd 2>/dev/null | \
    grep -oP '/proc/\d+' | sort -u
```

> *"I walk `/proc/PID/fd/` for every process. Each fd is a symlink — `readlink` gives the actual path. If it matches my target file, that process has it open. This is exactly what `lsof` does internally."*

---

### Q2. "What is `/proc/PID/fd/` and what does it contain?"

**Answer:**
> *"Every running process has a directory at `/proc/PID/fd/` containing symbolic links — one per open file descriptor. FD 0, 1, 2 are always stdin, stdout, stderr. Additional FDs point to whatever the process has opened: regular files, sockets, pipes, devices. Each `readlink` gives the actual path. This is the kernel's real-time map of what every process has open — it updates instantly as files are opened and closed. No tool needed, no installation required — it's part of the Linux kernel's proc filesystem."*

---

### Q3. "What does `/proc/PID/io` tell you about write activity?"

**Answer:**
> *"`write_bytes` in `/proc/PID/io` is the cumulative number of bytes that actually hit the disk for this process since it started. By taking two snapshots a few seconds apart and dividing by the interval, you get the current write rate. `wchar` is bytes written to the write syscalls including cache — it's higher than `write_bytes` because not all writes make it to physical disk. For log file investigation, `write_bytes` growing rapidly is the smoking gun — it confirms the process is flushing to disk, not just writing to a buffer."*

```bash
# Rate calculation:
W1=$(awk '/write_bytes/{print $2}' /proc/$PID/io); sleep 3
W2=$(awk '/write_bytes/{print $2}' /proc/$PID/io)
echo "Write rate: $(( (W2-W1)/3 )) bytes/sec"
```

---

### Q4. "What is the difference between `lsof` and reading `/proc/PID/fd`?"

**Answer:**
> *"Both give the same information — `lsof` is a wrapper around `/proc`. The key differences are: `lsof` formats output nicely, shows FD access mode (r/w/u for read/write/both), byte offset position, and file inode. `/proc/fd` scan is always available — no installation needed, works even on minimal systems. For scripting in production environments where you can't guarantee tools are installed, `/proc` is the reliable approach. `lsof` is better for interactive investigation; `/proc` scan is better for scripts and automation."*

---

### Q5. "How would you distinguish the direct writer vs the syslog source?"

**Answer:**
> *"Two separate questions. The direct writer — the process with an open fd to the file — is found via `/proc/PID/fd` or `lsof`. On most systems that's `rsyslogd`, which is the log daemon. But rsyslogd just forwards what OTHER processes send via `syslog()`. The real source of excessive messages is found by analyzing the log content itself — `sort | uniq -c | sort -rn` on the hostname or application field to see who is generating the most entries. Then check `logger -t` or `strace -e sendmsg -p $(pgrep rsyslogd)` to see incoming messages in real time."*

```bash
# Who is generating most log lines?
awk '{print $5}' /var/log/messages | sort | uniq -c | sort -rn | head -10
#        ↑ field 5 = process name/PID in standard syslog format
```

---

### Q6. "How would you monitor file writes in real time without polling?"

**Answer:**
> *"`inotifywait` from the `inotify-tools` package uses the Linux kernel's inotify API — event-driven, zero polling overhead. It registers interest in specific file events and the kernel notifies you instantly when they occur. For log monitoring:"*

```bash
# Watch for any write to the file:
inotifywait -m -e modify /var/log/messages

# Combine with /proc fd scan for attribution:
inotifywait -m -e modify /var/log/messages 2>/dev/null | \
while read dir event file; do
    echo "--- Write detected at $(date +%H:%M:%S) ---"
    for pid in /proc/[0-9]*/fd; do
        p=$(echo "$pid" | cut -d/ -f3)
        readlink ${pid}/* 2>/dev/null | grep -q "/var/log/messages" && \
            echo "  PID $p: $(tr '\0' ' ' < /proc/$p/cmdline 2>/dev/null)"
    done
done
```

---

### Q7. "What's `strace` and when would you use it for this investigation?"

**Answer:**
> *"`strace` intercepts and logs system calls made by a process in real time. For log write investigation, `strace -p PID -e trace=write,open` shows every write syscall with its arguments — including the actual content being written. It's more invasive than fd inspection (adds overhead, can slow the process) but gives much deeper insight: you can see exactly what text is being written at the moment it happens."*

```bash
# Trace writes to a specific fd number (fd 4 in this example):
strace -p 1299 -e trace=write -e write=4 2>&1 | head -20

# Trace all file opens by rsyslogd:
strace -p $(pgrep rsyslogd) -e trace=openat,write 2>&1 | grep messages

# See who is SENDING to syslog socket:
strace -p $(pgrep rsyslogd) -e trace=recvmsg 2>&1 | head -20
```

---

## Part 5: Cheat Sheet

```
FIND WRITER (no tools):
  for pid in /proc/[0-9]*/fd; do
    p=$(echo "$pid" | cut -d/ -f3)
    for fd in "$pid"/*; do
      [ "$(readlink "$fd" 2>/dev/null)" = "/var/log/messages" ] && \
        echo "PID $p ($(cat /proc/$p/comm))"
    done
  done

FIND WRITER (with lsof):
  lsof /var/log/messages
  lsof +D /var/log/           # all files in directory

CHECK WRITE RATE:
  W1=$(awk '/write_bytes/{print $2}' /proc/$PID/io)
  sleep 5
  W2=$(awk '/write_bytes/{print $2}' /proc/$PID/io)
  echo "$(( (W2-W1)/5 )) bytes/sec"

SAVE REPORT:
  { ps -p $PID -o pid,ppid,user,stat,pcpu,pmem,rss,lstart,cmd;
    echo "--- LAST 50 LINES ---";
    tail -50 /var/log/messages; } > /home/devops/excessive_log_process.txt

REAL-TIME MONITORING:
  inotifywait -m -e modify /var/log/messages   # event-driven
  watch -n 1 'ls -lh /var/log/messages'        # size polling
  tail -f /var/log/messages                    # follow content

/proc FILES USED:
  /proc/$PID/fd/*     → open file descriptors (symlinks)
  /proc/$PID/io       → cumulative I/O bytes (write_bytes key)
  /proc/$PID/comm     → process name
  /proc/$PID/cmdline  → full command (null-delimited)
  /proc/$PID/status   → UID, PPID, VmRSS, state

FIND LOG SOURCE (who sends most syslog messages):
  awk '{print $5}' /var/log/messages | sort | uniq -c | sort -rn | head -10

STRACE (deep inspection):
  strace -p $PID -e trace=write 2>&1 | grep messages
```

> **Bloomberg interview tip:** Bloomberg's trading systems have zero tolerance for disk saturation — a log flood can cause latency spikes that cost millions in microseconds. They'll push you on **prevention**: mention `rsyslog rate limiting` (`$imuxsock RateLimit.Interval 5` + `RateLimit.Burst 500`) to cap how many messages any process can send per second, and `logrotate --force` for immediate rotation when a file grows unexpectedly. Showing you think about preventing the incident, not just diagnosing it, is what differentiates senior candidates at Bloomberg.
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Track Forking Process Hierarchies
> **Company:** Splunk | **Difficulty:** Easy
---
#### **Scenario**
System resources are being consumed by an unusually large process tree. You need to identify the parent process with the most children and document its hierarchy.
#### **Task**
Identify the process tree with the highest number of child processes and save its hierarchy—including **PIDs** and **full command names (arguments)**—to: `/home/devops/process_tree_report.txt`.
Linux provides the `pstree` command to display full hierarchical process trees, including PIDs and command arguments.
#### **Example**
The file `/home/devops/process_tree_report.txt` should look similar to this:
```text
spawn_many_workers.sh,159 /home/devops/spawn_many_workers.sh
 ├─sleep,209 infinity
 ├─spawn_many_workers.sh,190 /home/devops/spawn_many_workers.sh
 │  └─sleep,220 60
 ├─spawn_many_workers.sh,191 /home/devops/spawn_many_workers.sh
 ...
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/track-forking-process-hierarchies)

Process tree captured. Now build the complete solution.

### Step 5 — Complete Solution: Find Top Parent + Save ReportReport saved. Output matches the required format.

---

## Part 3: Every Concept Explained Simply

### `pstree` — Every Flag Decoded

```bash
pstree -p -a $PID

pstree        →  show process TREE (parent → children hierarchy)
                 without flags: shows names only, collapses identical siblings

  -p          →  show PIDs in parentheses
                 python3(551) instead of just python3

  -a          →  show full command line ARGUMENTS
                 python3 /tmp/spawner2.py  instead of  python3

  $PID        →  start tree from this specific root
                 without: starts from PID 1 (entire system tree)

Output format:
  python3,551 /tmp/spawner2.py   ← root
    ├─python3,553 /tmp/spawner2.py   ← child
    ├─python3,554 /tmp/spawner2.py
    │   └─python3,566 /tmp/spawner2.py  ← grandchild
    └─python3,555 /tmp/spawner2.py
```

### How to Find "Most Children" Without pstree

```bash
# From ps output: count how many processes have each PPID
ps -eo pid,ppid --no-headers | \
    awk '{count[$2]++} END{
        for(p in count) print count[p], p
    }' | sort -rn | head -5

# Output:
# 49  2        ← kthreadd (kernel, ignore)
# 12  551      ← our spawner ← highest non-kernel
# 3   1        ← init/systemd

# Exclude kernel threads (PID < 100):
awk '{count[$2]++} END{
    for(p in count) if(p+0>100) print count[p], p
}' | sort -rn | head -1
```

### The PPID Chain — Reading Process Ancestry

```
Every process in /proc/PID/status has a PPid line:

cat /proc/566/status | grep -E "^(Name|Pid|PPid)"
  Name: python3
  Pid:  566
  PPid: 563       ← parent is 563

cat /proc/563/status | grep -E "^(Name|Pid|PPid)"
  Name: python3
  Pid:  563
  PPid: 551       ← grandparent is 551

cat /proc/551/status | grep -E "^(Name|Pid|PPid)"
  Name: python3
  Pid:  551
  PPid: 1         ← great-grandparent is init

So PID 566 ancestry chain: 566 → 563 → 551 → 1
pstree shows this as:
  python3,551
    └─python3,563
        └─python3,566  ← us
```

### `pstree -p` Output — Extracting PIDs Programmatically

```bash
pstree -p 551
# python3(551)─┬─python3(553)
#              ├─python3(554)
#              ├─python3(563)─┬─python3(565)
#              │              └─python3(567)
#              └─python3(564)─┬─python3(566)
#                             └─python3(568)

# Extract all PIDs from pstree output:
pstree -p 551 | grep -oP '\(\d+\)' | tr -d '()'
# 551 553 554 563 565 567 564 566 568

# Count total processes in tree:
pstree -p 551 | grep -oP '\(\d+\)' | wc -l
# 9
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How do you find the process with the most child processes?"

**Answer:**
```bash
# Count direct children per PPID:
ps -eo pid,ppid --no-headers | \
    awk '{c[$2]++; n[$2]=$3} END{
        for(p in c) if(p+0>100) print c[p], p
    }' | sort -rn | head -5

# One-liner to get just the top PID:
ps -eo ppid --no-headers | sort | uniq -c | sort -rn | \
    awk 'NR==1{print $2}'
```

> *"I count how many processes share each PPID — that's the direct child count per parent. I filter out kernel threads (PID < 100) since `kthreadd` always has many kernel worker children that aren't application processes. The process with the highest count is the top offender."*

---

### Q2. "What does `pstree -p -a` show and why use both flags?"

**Answer:**
> *"`pstree` by default shows just process names and collapses identical siblings — `10*[python3]` instead of listing each one. `-p` adds the PID in parentheses after each name, which is essential for investigation — you need PIDs to send signals, check `/proc`, or reference specific processes. `-a` shows the full command line with arguments — without it you only see `python3` not `python3 /home/devops/spawn_workers.sh`, losing critical context about what the process is actually doing. Together they give a complete picture: who is running what, with exact PIDs."*

---

### Q3. "What's the difference between direct children and total descendants?"

**Answer:**
> *"Direct children are processes whose PPID equals the parent's PID — immediate forks. Total descendants include children, grandchildren, great-grandchildren — everyone who can trace their ancestry back to the root. In `pstree`, direct children appear at the first indent level under the parent. For resource investigation, direct children matters for 'is this process forking excessively?' while total descendants matters for 'what's the total resource footprint of this entire process family?' A process that spawns 5 children who each spawn 20 grandchildren has 5 direct children but 105 total descendants."*

---

### Q4. "How would you kill an entire process tree including all descendants?"

**Answer:**
```bash
# Method 1: kill by process group (if all in same group)
kill -TERM -$PID    # negative PID = kill process group

# Method 2: pkill with -P (parent PID)
pkill -TERM -P $PID  # kill direct children only
# Recursive: need a loop

# Method 3: kill entire subtree recursively
kill_tree() {
    local pid=$1
    # Kill children first (bottom-up)
    for child in $(ps -o pid --no-headers --ppid $pid 2>/dev/null); do
        kill_tree $child
    done
    kill -TERM $pid 2>/dev/null
}
kill_tree $TOP_PID

# Method 4: use pstree to get all PIDs, then kill
pstree -p $TOP_PID | grep -oP '\(\d+\)' | tr -d '()' | \
    xargs kill -TERM 2>/dev/null

# Method 5: systemd/cgroups (cleanest — kills entire cgroup)
systemctl stop $SERVICE_NAME
```

---

### Q5. "What is `/proc/PID/status` and what fields are useful for process hierarchy investigation?"

**Answer:**
```bash
cat /proc/551/status

# Key fields for hierarchy investigation:
Name:   python3        ← process name (same as /proc/PID/comm)
Pid:    551            ← this process's PID
PPid:   1              ← PARENT PID ← trace ancestry here
TracerPid: 0           ← 0=not being traced, nonzero=debugger attached
Uid:    0 0 0 0        ← real/effective/saved/filesystem UID
Gid:    1000 1000...   ← group IDs
Threads: 1             ← thread count (>1 = multi-threaded)
VmRSS:  9688 kB       ← actual RAM in use
```

> *"`PPid` is the key field — it lets you traverse the ancestry chain manually without any tools. Follow PPid recursively until you reach PID 1 to understand the complete chain of processes that led to this one being spawned."*

---

### Q6. "How would you monitor for new process spawning in real time?"

**Answer:**
```bash
# Method 1: watch process count for a parent
watch -n 1 "ps --ppid $PID --no-headers | wc -l"

# Method 2: auditd (kernel-level fork monitoring)
auditctl -a always,exit -F arch=b64 -S fork,clone -k fork_monitor
ausearch -k fork_monitor --interpret | tail -20

# Method 3: eBPF / bpftrace (modern, low overhead)
bpftrace -e 'tracepoint:syscalls:sys_enter_fork { 
    printf("fork by PID %d (%s)\n", pid, comm); 
}'

# Method 4: inotifywait on /proc
# (watches for new /proc/NNNN directories = new processes)
inotifywait -m -e create /proc 2>/dev/null | \
    grep -oP '(?<= )\d+$' | \
    while read newpid; do
        comm=$(cat /proc/$newpid/comm 2>/dev/null)
        ppid=$(awk '/^PPid:/{print $2}' /proc/$newpid/status 2>/dev/null)
        echo "New PID $newpid ($comm) forked from $ppid"
    done
```

---

### Q7. "What does the STAT column mean in `ps` output for child processes?"

```
ps -o pid,stat,comm

STAT codes:
  S   ← Sleeping (waiting for event) — most processes
  R   ← Running (using CPU right now)
  D   ← Uninterruptible sleep (disk I/O wait)
  Z   ← Zombie (exited but not reaped) ← parent bug
  T   ← Stopped (SIGSTOP or debugger)

Second character modifiers:
  s   ← session leader (started a new session, e.g. shell)
  +   ← in foreground process group
  l   ← multi-threaded (has multiple threads)
  <   ← high priority (negative nice value)
  N   ← low priority (positive nice value)

For spawned worker processes: Ss or S+ is normal
Z   = parent has bug (not calling wait()) — zombie accumulation
D+  = many processes blocked on I/O = disk bottleneck
```

---

### Q8. "How would you automatically alert when any process exceeds N children?"

**Answer:**
```bash
#!/bin/bash
# /usr/local/bin/fork_monitor.sh
THRESHOLD=20
INTERVAL=30
ALERT_EMAIL="ops@splunk.com"

while true; do
    # Find any process with more children than threshold
    OFFENDER=$(ps -eo pid,ppid --no-headers | \
        awk '{c[$2]++; n[$2]=$1} END{
            for(p in c) if(c[p]>='$THRESHOLD' && p+0>100) print c[p], p
        }' | sort -rn | head -1)

    if [ -n "$OFFENDER" ]; then
        COUNT=$(echo $OFFENDER | awk '{print $1}')
        PID=$(echo $OFFENDER | awk '{print $2}')
        CMD=$(tr '\0' ' ' < /proc/$PID/cmdline 2>/dev/null)
        MSG="ALERT: PID $PID ($CMD) has $COUNT children (threshold: $THRESHOLD)"
        echo "$MSG"
        # Save tree snapshot immediately for forensics
        pstree -p -a $PID > /tmp/fork_alert_$(date +%s).txt
        echo "$MSG" | mail -s "Fork bomb alert on $(hostname)" "$ALERT_EMAIL"
    fi

    sleep $INTERVAL
done
```

---

## Part 5: Cheat Sheet

```
KEY PSTREE FLAGS:
  pstree $PID          → tree from specific root (names only)
  pstree -p $PID       → include PIDs: python3(551)
  pstree -a $PID       → include full cmdline args
  pstree -p -a $PID    → both — BEST FOR INVESTIGATION
  pstree -s $PID       → show parents (path to root)
  pstree -n $PID       → sort by PID (numeric order)
  pstree -c $PID       → don't compact identical branches

EXTRACT PIDs FROM pstree OUTPUT:
  pstree -p $PID | grep -oP '\(\d+\)' | tr -d '()'
  pstree -p $PID | grep -oP '\(\d+\)' | wc -l  # count

FIND TOP PARENT BY CHILD COUNT:
  ps -eo pid,ppid --no-headers | \
    awk '{c[$2]++} END{for(p in c) if(p>100) print c[p],p}' | \
    sort -rn | head -1

COUNT DIRECT CHILDREN OF A PID:
  ps --ppid $PID --no-headers | wc -l
  ps -eo ppid | grep -c "^ *${PID}$"

FULL ANCESTRY CHAIN (bottom up):
  pid=$PID
  while [ "$pid" -gt 1 ]; do
    printf "PID %s: %s\n" "$pid" "$(cat /proc/$pid/comm)"
    pid=$(awk '/^PPid:/{print $2}' /proc/$pid/status)
  done

KILL ENTIRE TREE:
  pstree -p $PID | grep -oP '\d+' | xargs kill -TERM
  kill -TERM -$PID   # if same process group

SAVE REPORT:
  { pstree -p -a $PID; echo "---";
    ps --ppid $PID -o pid,user,stat,cmd; } \
    > /home/devops/process_tree_report.txt
```

> **Splunk interview tip:** Splunk ingests and analyzes machine data at massive scale — they'll love if you mention that `pstree` output can be shipped directly into Splunk for process lineage analysis, and that eBPF tools like `bpftrace` give real-time fork tracing with near-zero overhead. Mentioning that fork bombs are best prevented with `ulimit -u <max_procs>` per user or `TasksMax=` in systemd unit files shows you think prevention, not just diagnosis.


+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Update AWS Configs
Company: Stripe | Difficulty: Medium
Scenario
Each application environment (staging, dev, prod) has its own configuration file stored under `/etc/app/envs/`, and each file currently has `multi_az = false` and `availability_zone = "us-east-1a"`. Manually editing each file is error-prone and inefficient, so the change must be automated.
Task
Locate all `.conf` files under `/etc/app/envs/` across different environment subdirectories, update the `multi_az` setting from `false` to `true`, modify the `availability_zone` line to include two zones `"us-east-1a,us-east-1b"`, perform these edits in-place while preserving all other configuration values.
Example

```
# Before (single-AZ configuration)

region = "us-east-1"
availability_zone = "us-east-1a"
multi_az = false

```


```
# After (multi-AZ configuration enabled)

region = "us-east-1"
availability_zone = "us-east-1a,us-east-1b"
multi_az = true

```

📹 [Video Solution](https://prepare.sh/interview/devops/terminal/update-aws-configs)


## Update AWS Configs — Full Deep Dive

---

## Part 1: Understand It Simply

### What Are We Doing and Why?

```
Before migration (single-AZ — risk of outage):
  /etc/app/envs/staging/app.conf   ← multi_az = false
  /etc/app/envs/dev/app.conf       ← multi_az = false
  /etc/app/envs/prod/app.conf      ← multi_az = false

Manual edit = dangerous:
  3 files today → 30 files next month
  Miss one → prod still single-AZ → outage
  Typo → broken config → deploy fails

Automated sed = safe and repeatable:
  find all .conf files → apply same edit to ALL → done
  Run again = idempotent (already true stays true)
```

### The Two Target Lines

```
Line 1: multi_az = false          →  multi_az = true
Line 2: availability_zone = "us-east-1a"  →  availability_zone = "us-east-1a,us-east-1b"

sed pattern for line 1:
  s/multi_az = false/multi_az = true/

sed pattern for line 2:
  s/availability_zone = "us-east-1a"/availability_zone = "us-east-1a,us-east-1b"/

Key requirement: in-place (-i flag)
  Without -i: sed prints to stdout, file unchanged
  With -i:    sed edits the file directly on disk
```

---

## Part 2: Build and Execute

### Step 1 — Create the Environment### Step 2 — Understand `sed` Before Running### Step 3 — Backup First, Then Apply In-Place### Step 4 — Verify: Before vs AfterAll 5 files updated. All other values untouched. Now the deep explanation.

---

## Part 3: Every Concept Explained Simply

### `sed -i` — In-Place Editing Explained

```
Without -i (default):   sed reads file → processes → writes to STDOUT
                        Original file: UNCHANGED
                        Terminal output: changed version

With -i (in-place):     sed reads file → processes → writes BACK to file
                        Original file: CHANGED
                        Terminal output: nothing (silent)

With -i.bak (GNU sed):  sed reads file → writes changed to file
                        → saves ORIGINAL to file.bak automatically
                        (one command = edit + backup)

# macOS sed vs GNU sed difference:
  GNU sed (Linux): sed -i 's/old/new/' file    ← works
  macOS sed (BSD): sed -i '' 's/old/new/' file ← needs empty string
```

### `sed -e` — Multiple Expressions in One Pass

```bash
# Two separate sed calls (TWO passes through file):
sed -i 's/multi_az = false/multi_az = true/' file
sed -i 's/availability_zone = "us-east-1a"/.../' file

# One sed call with -e (ONE pass through file — faster):
sed -i \
    -e 's/multi_az = false/multi_az = true/' \
    -e 's/availability_zone = "us-east-1a"/.../' \
    file

# Equivalently, using semicolon separator:
sed -i 's/multi_az = false/multi_az = true/; s/availability_zone.../.../' file

# When does ONE pass matter?
# When both patterns could match the same line
# With two passes: second sed runs on OUTPUT of first
# With one pass:   both substitutions run on ORIGINAL line
```

### The `find -exec {} +` vs `-exec {} \;`

```bash
# {} \;  — runs command ONCE PER FILE
find /etc/app/envs/ -name "*.conf" -exec sed -i 's/x/y/' {} \;
# Spawns: sed file1, sed file2, sed file3 ...
# N files = N processes = slower

# {} +   — batches files together (like xargs)
find /etc/app/envs/ -name "*.conf" -exec sed -i 's/x/y/' {} +
# Runs: sed file1 file2 file3 ... (all in one call!)
# Much faster for many files

# BUT for sed -i with multiple files, {} + is correct:
# sed -i 's/x/y/' file1 file2 file3  ← works ✓
# sed IS designed to handle multiple files
```

### Why `! -name "*.bak"` Protects Backups

```
Without the exclusion:
  find /etc/app/envs/ -name "*.conf"
  → /etc/app/envs/prod/app.conf
  → /etc/app/envs/prod/app.conf.bak   ← oops, found the backup too!
  sed edits BOTH → backup loses its "before" state

With exclusion:
  find /etc/app/envs/ -name "*.conf" ! -name "*.bak"
  → /etc/app/envs/prod/app.conf        ← edited ✓
  backup stays untouched               ← preserved ✓

  ! -name "*.bak" = NOT matching *.bak
  (the ! is a logical NOT for the next condition)
```

### Anchoring Patterns to Avoid Wrong Matches

```bash
# Naive pattern (risky):
sed 's/false/true/'
# Would also change:
# "false_positive = 'false'" → "true_positive = 'true'"  ← WRONG!

# Better: match the specific key=value pair
sed 's/multi_az = false/multi_az = true/'
# Only matches this exact line — other "false" values untouched ✓

# Even better: anchor to start of line with ^
sed 's/^multi_az = false/multi_az = true/'
# ^ = must be at line beginning
# Protects against: "  multi_az = false" (with leading space) — won't match
# (depends on whether your config has indentation)

# Most robust: use word boundaries or exact key match
sed 's/^\(multi_az\s*=\s*\)false$/\1true/'
# \(...\) = capture group (basic regex)
# \1 = back-reference to capture group
# Handles: multi_az=false, multi_az = false, multi_az  =  false
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "What does `sed -i` do? What's the difference from regular `sed`?"

**Answer:**
> *"`sed` by default reads input and writes processed output to stdout — the original file is untouched. The `-i` flag changes that to in-place mode: sed writes the result back into the original file, effectively editing it directly on disk. Some teams use `sed -i.bak` to simultaneously create a backup — GNU sed creates `file.bak` containing the original before overwriting `file` with the changes. The in-place flag is essential for automating config updates across many files since you can't pipe stdout back to the same file you're reading."*

---

### Q2. "Why use `find -exec sed -i {} +` instead of a `while read` loop?"

**Answer:**
> *"The `{}+` form batches all found files into a single sed invocation — `sed -i 's/x/y/' file1 file2 file3`. A `while read` loop calls sed once per file, spawning a new process for every iteration. For 3 files it doesn't matter; for 3,000 files, the difference is significant. The `while read` loop has one advantage: you can add per-file logging — `echo Updated: $f` — and handle per-file errors. For a simple mass substitution, `{}+` is cleaner and faster. Both are correct; the choice depends on whether you need per-file control."*

---

### Q3. "How would you make the sed pattern more robust to handle variations like `multi_az=false` (no spaces)?"

**Answer:**
> *"Use `\s*` to match zero or more whitespace characters around the `=`:"*

```bash
# Handles: "multi_az=false", "multi_az = false", "multi_az  =  false"
sed -i 's/^multi_az\s*=\s*false/multi_az = true/' file

# OR use extended regex with -E:
sed -i -E 's/^multi_az\s*=\s*false/multi_az = true/' file

# Even more robust — preserve original spacing:
sed -i -E 's/^(multi_az\s*=\s*)false/\1true/' file
# \1 captures "multi_az = " and replaces only "false" with "true"
# preserving whatever whitespace was around the =
```

---

### Q4. "How would you verify the changes are correct without modifying files?"

**Answer:**
> *"Never run sed -i without a dry run first. Use `sed` without `-i` to preview, then `grep` to verify after applying:"*

```bash
# DRY RUN — see what would change (no -i):
find /etc/app/envs/ -name "*.conf" ! -name "*.bak" | while read f; do
    echo "=== $f ==="
    sed -n 's/multi_az = false/multi_az = true/p' "$f"
    # -n suppresses default output
    # p at end of s command prints only MATCHED lines
done

# POST-CHANGE verification:
grep -r "multi_az" /etc/app/envs/ --include="*.conf" | grep -v ".bak"
# Every line should show: multi_az = true

# Verify nothing unwanted changed:
diff /etc/app/envs/prod/app.conf.bak /etc/app/envs/prod/app.conf
# Should show exactly 2 changed lines
```

---

### Q5. "How do you handle the backup automatically within `sed -i`?"

**Answer:**
> *"GNU sed accepts a suffix after `-i` to create an automatic backup: `sed -i.bak 's/x/y/' file` edits `file` in-place and saves the original as `file.bak`. This is a single atomic operation — you get the edit and the backup in one command without a separate `cp` step:"*

```bash
# Creates .bak automatically for every file:
find /etc/app/envs/ -name "*.conf" -type f \
    -exec sed -i.bak \
        -e 's/multi_az = false/multi_az = true/' \
        -e 's/availability_zone = "us-east-1a"/availability_zone = "us-east-1a,us-east-1b"/' \
    {} +

# Verify backups were created:
find /etc/app/envs/ -name "*.conf.bak"

# Restore a single file from backup:
cp /etc/app/envs/prod/app.conf.bak /etc/app/envs/prod/app.conf
```

---

### Q6. "How would you roll back the changes if something goes wrong?"

**Answer:**
```bash
# Restore all files from backups:
find /etc/app/envs/ -name "*.conf.bak" | while IFS= read -r bak; do
    original="${bak%.bak}"
    cp "$bak" "$original"
    echo "Restored: $original"
done

# OR use sed to reverse the specific changes:
find /etc/app/envs/ -name "*.conf" ! -name "*.bak" \
    -exec sed -i \
        -e 's/multi_az = true/multi_az = false/' \
        -e 's/availability_zone = "us-east-1a,us-east-1b"/availability_zone = "us-east-1a"/' \
    {} +

# Verify rollback:
grep -r "multi_az" /etc/app/envs/ --include="*.conf" | grep -v ".bak"
# Should show: multi_az = false
```

---

### Q7. "What's the difference between `sed` and `awk` for this task? When would you use `awk`?"

**Answer:**
> *"`sed` is perfect for simple line-by-line substitutions — find a pattern, replace it. `awk` is better when you need to make changes based on field values, conditionally update only certain keys, or process structured data. For this config format, `awk` gives more control:"*

```bash
# awk approach — same result, more flexible:
awk '
    /^multi_az =/ { sub(/false/, "true") }
    /^availability_zone =/ { sub(/"us-east-1a"/, "\"us-east-1a,us-east-1b\"") }
    { print }
' /etc/app/envs/prod/app.conf

# awk advantage: conditional logic
awk '
    /^multi_az =/ && /false/ { sub(/false/, "true") }  # only if currently false
    /^availability_zone =/ && !/,/ { sub(/"([^"]+)"/, "\"&,us-east-1b\"") }  # only if single AZ
    { print }
' file

# awk for in-place (use a temp file or GNU awk):
awk '...' file > file.tmp && mv file.tmp file
# OR with GNU awk: gawk -i inplace '...' file
```

---

## Part 5: Cheat Sheet

```
CORE COMMAND:
  find /etc/app/envs/ -name "*.conf" ! -name "*.bak" -type f \
      -exec sed -i \
          -e 's/multi_az = false/multi_az = true/' \
          -e 's/availability_zone = "us-east-1a"/availability_zone = "us-east-1a,us-east-1b"/' \
      {} +

BACKUP FIRST:
  find /etc/app/envs/ -name "*.conf" -exec cp -p {} {}.bak \;
  # OR auto-backup in sed:
  sed -i.bak 's/old/new/' file

DRY RUN (preview only):
  sed -n 's/multi_az = false/multi_az = true/p' file
  # -n = suppress normal output
  # p  = print only substituted lines

VERIFY AFTER:
  grep -r "multi_az" /etc/app/envs/ --include="*.conf" | grep -v ".bak"
  grep -rL "us-east-1b" /etc/app/envs/ --include="*.conf"  # files WITHOUT update
  diff file.bak file                                          # what changed

ROLLBACK:
  find /etc/app/envs/ -name "*.bak" | while read b; do cp "$b" "${b%.bak}"; done

sed FLAGS:
  -i         → in-place (edit file directly)
  -i.bak     → in-place + auto backup original
  -e         → multiple expressions in one call
  -n         → suppress default output (use with p flag)
  -E or -r   → extended regex (enables + ? | without escaping)

PATTERN ROBUSTNESS:
  s/false/true/           → naive, may hit wrong lines
  s/multi_az = false/.../  → better, specific key
  s/^multi_az = false/.../ → best, anchored to line start
  s/^(key\s*=\s*)false/\1true/  → preserves original whitespace

find + exec:
  -exec cmd {} \;   → one process per file (slower, more control)
  -exec cmd {} +    → batch files together (faster, like xargs)
  ! -name "*.bak"   → exclude backup files from processing
```

> **Stripe interview tip:** Stripe's infrastructure spans many environments with strict config management. Always mention **idempotency** — running the sed command twice should produce the same result (already-updated `true` won't match `false` pattern). Also bring up **validation after the change** — at Stripe scale you'd run a diff against a known-good template and check config syntax before deploying, not just trust that sed ran without errors. That systematic verification mindset is exactly what Stripe's reliability engineering culture values.

+++++++++++++++++++++++++++++++++++++++++++++++++++++


# Upload Safe File Partitioning
> **Company:** GoDaddy | **Difficulty:** Medium
---
#### **Scenario**
Your application uploads files from `/tmp/app/`, but the maximum allowed file size is 1 MB, and some files exceed this limit.
#### **Task**
Find all files larger than 1 MB in `/tmp/app/` and its subdirectories, split each oversized file into 1 MB chunks in the same directory where the original file is located with a recognizable naming pattern (e.g., original_filename.part_aa), keep the original files intact, and verify that the chunks were created successfully.
#### **Example**
```
# Before (files exceed 1 MB limit)
/tmp/app/uploads/video.mp4 (3.2 MB)
/tmp/app/data/archive.tar.gz (2.5 MB)
Cannot upload due to size restrictions
```
```
# After (files split into 1 MB chunks)
/tmp/app/uploads/video.mp4
/tmp/app/uploads/video.mp4.part_aa
/tmp/app/uploads/video.mp4.part_ab
/tmp/app/uploads/video.mp4.part_ac
/tmp/app/data/archive.tar.gz
/tmp/app/data/archive.tar.gz.part_aa
/tmp/app/data/archive.tar.gz.part_ab
Chunks ready for upload within size limits
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/upload-safe-file-partitioning)



## Upload Safe File Partitioning — Full Deep Dive

---

## Part 1: Understand It Simply

### What Are We Doing and Why?

```
Upload API limit: 1 MB per file

Problem:
  video.mp4     = 3.2 MB  → upload REJECTED ✗
  archive.tar.gz = 2.5 MB → upload REJECTED ✗

Solution: split into 1 MB chunks, upload each chunk separately

  video.mp4.part_aa = 1 MB  → upload ✓
  video.mp4.part_ab = 1 MB  → upload ✓
  video.mp4.part_ac = 1 MB  → upload ✓
  video.mp4.part_ad = 0.2 MB → upload ✓ (remainder)

On the receiving side: reassemble with:
  cat video.mp4.part_* > video.mp4
```

### The `split` Command Naming System

```
split -b 1M --additional-suffix=".part_" file.mp4 "file.mp4."

Generates:
  file.mp4.aa    ← auto suffix
  file.mp4.ab
  file.mp4.ac

With --additional-suffix=".part_" added AFTER the auto suffix:
  Wait — that's not right. Let me clarify the naming:

split -b 1M file.mp4 "file.mp4.part_"

The PREFIX is:     "file.mp4.part_"
The AUTO-SUFFIX is: aa, ab, ac ...

Result:
  file.mp4.part_aa  ← PREFIX + auto suffix ✓
  file.mp4.part_ab
  file.mp4.part_ac
```

### The Directory Challenge

```
find returns full paths:
  /tmp/app/uploads/video.mp4
  /tmp/app/data/archive.tar.gz

split needs:
  OUTPUT PREFIX = same directory + filename + .part_
  /tmp/app/uploads/video.mp4.part_

We build this with:
  DIRNAME  = $(dirname  "/tmp/app/uploads/video.mp4") = "/tmp/app/uploads"
  BASENAME = $(basename "/tmp/app/uploads/video.mp4") = "video.mp4"
  PREFIX   = "${DIRNAME}/${BASENAME}.part_"
           = "/tmp/app/uploads/video.mp4.part_"
```

---

## Part 2: Build and Execute

### Step 1 — Create the Test Environment### Step 2 — Understand `split` Completely### Step 3 — The Core Solution### Step 4 — Verify EverythingAll files split correctly. MD5 checksums match — zero data loss. Now the deep explanation.

---

## Part 3: Every Concept Explained Simply

### `split -b 1M` — Byte-Based Splitting in Detail

```
split -b 1M input.file output_prefix

-b SIZE   = split by BYTES
  1M      = 1,048,576 bytes exactly (binary MB)
  1MB     = same (split accepts M and MB)
  1000000 = 1,000,000 bytes (decimal MB — avoid ambiguity)

How it works:
  Reads input.file sequentially
  Writes 1,048,576 bytes to output_prefix + aa
  Writes next 1,048,576 bytes to output_prefix + ab
  Writes remainder to output_prefix + ac (may be smaller)
  Stops when input is exhausted

Original file: NEVER modified (read-only operation)
```

### The Naming System: `PREFIX` + Auto-Suffix

```
split -b 1M video.mp4 "video.mp4.part_"
               ↑             ↑
           INPUT FILE     OUTPUT PREFIX

PREFIX = everything before the auto-generated suffix

Auto-suffix (default -a 2):
  aa  → first chunk
  ab  → second chunk
  ...
  az  → 26th chunk
  ba  → 27th chunk
  ...
  zz  → 676th chunk (maximum with 2-char suffix)

Result:
  video.mp4.part_aa    ← PREFIX "video.mp4.part_" + auto "aa"
  video.mp4.part_ab    ← PREFIX "video.mp4.part_" + auto "ab"
  video.mp4.part_ac    ← PREFIX "video.mp4.part_" + auto "ac"

For files needing MORE than 676 chunks (>676 MB):
  split -b 1M -a 3 video.mp4 "video.mp4.part_"
  → part_aaa, part_aab ... part_zzz (17,576 chunks = 17.5 GB)
```

### `dirname` and `basename` — Path Decomposition

```bash
f="/tmp/app/uploads/video.mp4"

dirname  "$f"  →  /tmp/app/uploads    ← directory component
basename "$f"  →  video.mp4           ← filename component

# Build the output prefix:
DIR=$(dirname "$f")                   # /tmp/app/uploads
BASE=$(basename "$f")                 # video.mp4
PREFIX="${DIR}/${BASE}.part_"         # /tmp/app/uploads/video.mp4.part_

# Why this matters:
# split "video.mp4" "video.mp4.part_"  ← creates chunks in CURRENT directory
# split "video.mp4" "/tmp/app/uploads/video.mp4.part_"  ← creates in SAME dir ✓
```

### `find -size +1M` — Size Filter

```bash
find /tmp/app -type f -size +1M

+1M  = strictly GREATER THAN 1MB
-1M  = strictly LESS THAN 1MB
 1M  = EXACTLY 1MB (rounded to nearest block)

# Note: find uses 1M = 1,048,576 bytes
# +1M means > 1,048,576 bytes → splits files that need splitting

# What about exactly 1MB files?
# -size +1M EXCLUDES exactly 1MB (they're fine as-is)
# Use -size +1024k if you want > 1024 KB (same threshold)

# Exclude chunks from being re-split:
find /tmp/app -type f -size +1M ! -name "*.part_*"
# ! -name "*.part_*" = NOT matching the chunk pattern
# Prevents splitting chunks of chunks if script runs twice
```

### MD5 Integrity Verification

```bash
# Original MD5:
md5sum video.mp4
→ a3f5b2e9c7d1... video.mp4

# Reassemble from chunks and compute MD5:
cat video.mp4.part_* | md5sum
→ a3f5b2e9c7d1...   ← SAME HASH = identical content ✓

# Why cat video.mp4.part_* works:
# Shell glob sorts alphabetically: part_aa, part_ab, part_ac
# cat streams them in order → reconstructs original byte sequence
# MD5 of stream = MD5 of original = data integrity confirmed

# For binary files (more robust than diff):
cmp video.mp4 <(cat video.mp4.part_*)
# cmp exits 0 if identical, non-zero if different
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "Walk me through how you find and split oversized files."

**Answer:**
```bash
find /tmp/app -type f -size +1M ! -name "*.part_*" | \
while IFS= read -r f; do
    DIR=$(dirname "$f")
    BASE=$(basename "$f")
    split -b 1M "$f" "${DIR}/${BASE}.part_"
done
```

> *"`find -type f -size +1M` locates regular files strictly over 1 MB. I exclude `*.part_*` so the script is idempotent — running it twice won't re-split existing chunks. For each file, `dirname` and `basename` decompose the path so `split` outputs chunks in the same directory as the original. `split -b 1M` creates exactly 1 MB chunks with alphabetical suffixes. The original is never modified — split only reads it."*

---

### Q2. "How does `split` name the output files? What's the maximum number of chunks?"

**Answer:**
> *"split appends a generated suffix to whatever prefix you provide. With the default `-a 2`, the suffix cycles through `aa`, `ab`, ..., `az`, `ba`, ..., `zz` — giving 26² = 676 possible chunks. At 1 MB per chunk, that handles files up to 676 MB. For larger files, use `-a 3` to get `aaa`–`zzz` = 17,576 chunks, handling up to ~17.5 GB. For even larger files or human-readable ordering, use `-d` for numeric suffixes: `00`, `01`, `02` — though alphabetic sorts correctly by default, numeric is more intuitive."*

---

### Q3. "How do you verify the chunks are complete and uncorrupted?"

**Answer:**
> *"Two levels of verification. First, check all chunks are within the size limit:"*

```bash
find /tmp/app -name "*.part_*" -size +1M
# Should return nothing — all chunks under 1MB ✓
```

> *"Second, and more importantly, verify data integrity with MD5 checksum:"*

```bash
ORIG_MD5=$(md5sum "$f" | awk '{print $1}')
RECON_MD5=$(cat "${f}.part_"* | md5sum | awk '{print $1}')
[ "$ORIG_MD5" = "$RECON_MD5" ] && echo "✓ Intact" || echo "✗ Corrupted"
```

> *"`cat part_*` relies on alphabetical glob ordering — `part_aa` before `part_ab` — which matches the split order, so the reassembled stream is byte-for-byte identical to the original."*

---

### Q4. "What does `! -name '*.part_*'` do in the find command?"

**Answer:**
> *"The `!` operator negates the following condition. `! -name '*.part_*'` means 'exclude files whose name matches `*.part_*`'. Without this, if you run the script twice, it would try to split the existing chunks — `video.mp4.part_aa` is still over nothing (it's 1MB exactly, so `-size +1M` wouldn't catch it), but a more defensive pattern is still good practice. It also makes the script idempotent: if something fails partway through and you re-run it, existing chunks won't cause confusion. The pattern `*.part_*` uses two wildcards — any name containing `.part_` anywhere."*

---

### Q5. "How would you reassemble the chunks after uploading?"

**Answer:**
```bash
# Reassemble a single file:
cat /tmp/app/uploads/video.mp4.part_* > /tmp/reassembled/video.mp4

# Verify integrity after reassembly:
md5sum /tmp/app/uploads/video.mp4
md5sum /tmp/reassembled/video.mp4
# Both should match ✓

# Reassemble all split files in a directory:
find /tmp/app -name "*.part_aa" | while IFS= read -r first_chunk; do
    # Get base name: remove .part_aa to get original name
    original="${first_chunk%.part_aa}"
    output_dir=$(dirname "$original")
    output_name=$(basename "$original")

    echo "Reassembling: $output_name"
    cat "${original}.part_"* > "/tmp/reassembled/${output_name}"
done
```

---

### Q6. "How would you handle files so large they need more than 676 chunks?"

**Answer:**
> *"Increase the suffix length with `-a N`. Each additional character multiplies capacity by 26:"*

```bash
split -b 1M -a 3 bigfile.iso "${DIR}/${BASE}.part_"
# -a 3 = 3-character suffix: aaa → zzz
# 26³ = 17,576 chunks × 1MB = 17.5 GB maximum

split -b 1M -a 4 hugefile.iso "${DIR}/${BASE}.part_"
# 26⁴ = 456,976 chunks × 1MB = ~447 GB maximum

# OR use numeric suffix (-d) which scales better:
split -b 1M -d -a 5 file "${DIR}/${BASE}.part_"
# 00000 → 99999 = 100,000 chunks × 1MB = ~97 GB
# Numeric is more human-readable and sorts correctly without special handling
```

---

### Q7. "What's the difference between `-b 1M` and `-b 1000000`? Which is correct for GoDaddy's 1 MB limit?"

**Answer:**
> *"1M in split means 1 mebibyte = 1,048,576 bytes (binary). 1000000 means exactly one million bytes (decimal). Cloud storage APIs and most upload limits use decimal MB (1,000,000 bytes). If GoDaddy's limit is 1,000,000 bytes exactly, using `-b 1M` would create chunks of 1,048,576 bytes — slightly over the limit! The safe approach:"*

```bash
# If limit is 1,000,000 bytes (decimal MB):
split -b 1000000 "$f" "${DIR}/${BASE}.part_"

# If limit is 1,048,576 bytes (binary MB):
split -b 1M "$f" "${DIR}/${BASE}.part_"

# To be safe: use slightly under the limit
split -b 950000 "$f" "${DIR}/${BASE}.part_"
# 950 KB chunks → always under both decimal and binary 1 MB
```

---

### Q8. "How would you make this script production-grade with logging and error handling?"

**Answer:**
```bash
#!/bin/bash
# upload_partition.sh — Production-grade file splitter

set -euo pipefail

SOURCE_DIR="${1:-/tmp/app}"
CHUNK_SIZE="${2:-1M}"
LOG="/var/log/file_partition.log"
ERRORS=0
PROCESSED=0

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG"; }

log "Starting partition run: dir=$SOURCE_DIR chunk_size=$CHUNK_SIZE"

find "$SOURCE_DIR" -type f -size +1M ! -name "*.part_*" | \
while IFS= read -r f; do
    DIR=$(dirname "$f")
    BASE=$(basename "$f")
    PREFIX="${DIR}/${BASE}.part_"
    ORIG_SIZE=$(du -sh "$f" | cut -f1)

    log "Processing: $f ($ORIG_SIZE)"

    if split -b "$CHUNK_SIZE" "$f" "$PREFIX" 2>>"$LOG"; then
        CHUNKS=$(ls "${PREFIX}"* 2>/dev/null | wc -l)
        # Integrity check
        ORIG_MD5=$(md5sum "$f" | awk '{print $1}')
        RECON_MD5=$(cat "${PREFIX}"* | md5sum | awk '{print $1}')
        if [ "$ORIG_MD5" = "$RECON_MD5" ]; then
            log "  ✓ OK: $CHUNKS chunks, MD5 verified"
            PROCESSED=$((PROCESSED + 1))
        else
            log "  ✗ INTEGRITY FAIL: $f — chunks deleted for safety"
            rm -f "${PREFIX}"*
            ERRORS=$((ERRORS + 1))
        fi
    else
        log "  ✗ SPLIT FAILED: $f"
        ERRORS=$((ERRORS + 1))
    fi
done

log "Done. Processed: $PROCESSED | Errors: $ERRORS"
[ "$ERRORS" -gt 0 ] && exit 1 || exit 0
```

---

## Part 5: Cheat Sheet

```
CORE COMMAND:
  find /tmp/app -type f -size +1M ! -name "*.part_*" | \
  while IFS= read -r f; do
      split -b 1M "$f" "$(dirname "$f")/$(basename "$f").part_"
  done

KEY split FLAGS:
  -b SIZE   → split by bytes: 1M, 1000000, 512K, 1G
  -l N      → split by N lines (text files)
  -a N      → suffix length (default 2 = aa..zz = 676 max)
  -d        → numeric suffix (00, 01, 02...)
  -n N      → split into exactly N equal chunks

SUFFIX CAPACITY:
  -a 2  →    676 chunks (default, up to 676 MB at 1MB/chunk)
  -a 3  → 17,576 chunks (up to 17.5 GB)
  -a 4  → 456,976 chunks (up to ~447 GB)

PATH DECOMPOSITION:
  dirname  "$f"  → /tmp/app/uploads (directory)
  basename "$f"  → video.mp4 (filename)

IDEMPOTENT FIND (exclude existing chunks):
  find /tmp/app -type f -size +1M ! -name "*.part_*"

VERIFY CHUNK SIZES:
  find /tmp/app -name "*.part_*" -size +1M  # should return nothing

MD5 INTEGRITY CHECK:
  ORIG=$(md5sum "$f" | awk '{print $1}')
  RECON=$(cat "${f}.part_"* | md5sum | awk '{print $1}')
  [ "$ORIG" = "$RECON" ] && echo "OK" || echo "FAIL"

REASSEMBLE:
  cat video.mp4.part_* > video.mp4  # glob sorts aa→ab→ac correctly

DECIMAL vs BINARY:
  1M  = 1,048,576 bytes (binary mebibyte)
  1MB = same in split
  1000000 = 1,000,000 bytes (decimal megabyte)
  Use 1000000 if API limit is decimal MB to avoid exceeding
```

> **GoDaddy interview tip:** GoDaddy handles massive file uploads for website assets, domain certificates, and hosting files. They'll ask about **resumability** — if a chunk upload fails at part 3 of 5, can you retry just that chunk? Mention that the part naming (`part_aa`, `part_ab`) makes this trivial — track which parts were ACKed by the server and retry only the failed ones. Also bring up `md5sum` checksums per chunk as API parameters for server-side verification — that's the production pattern their infrastructure team cares about.

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
# Uptime and Load Average Audit
> **Company:** Microsoft | **Difficulty:** Easy
---
#### **Scenario**
A Linux server is under review for stability and recent performance evaluation.
#### **Task**
Use the `uptime` command to determine how long the server has been running and extract the 15-minute load average as a floating-point number (e.g., 0.45). Save the uptime to `/home/devops/uptime.txt` and the 15-minute load average to `/home/devops/loadavg.txt`.
#### **Example**
```
Contents of `/home/devops/uptime.txt`: `2 days, 5:37`
Contents of `/home/devops/loadavg.txt`: `0.45`
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/uptime-and-load-average-audit)


## Uptime and Load Average Audit — Full Deep Dive

---

## Part 1: Understand It Simply

### What `uptime` Shows

```bash
$ uptime
 14:35:22 up 2 days, 5:37,  3 users,  load average: 0.12, 0.45, 0.38
 ├──────┘ ├──────────────┘  ├───────┘  ├───────────────────────────┘
 │         │                 │           │
 time now  uptime duration   users       1min, 5min, 15min load averages
           └── WANT THIS                                   └── WANT THIS
```

### What Is Load Average?

```
Load average = number of processes WAITING for CPU or disk at a given moment
               (averaged over the last 1, 5, or 15 minutes)

Single-core CPU examples:
  Load = 0.5  → CPU is 50% busy (healthy)
  Load = 1.0  → CPU is exactly at capacity
  Load = 2.0  → twice as many tasks as CPU can handle (overloaded)

Multi-core (4 cores):
  Load = 4.0  → all 4 cores busy (at capacity)
  Load = 8.0  → overloaded (twice capacity)

Rule of thumb:
  load / cpu_cores < 1.0  → healthy
  load / cpu_cores > 1.0  → investigate
  load / cpu_cores > 2.0  → serious problem

Why 15-minute average for auditing?
  1-min  → very recent spike, may be temporary
  5-min  → medium-term trend
  15-min → sustained load → TRUE picture of server health ✓
```

---

## Part 2: Build and Execute

### Step 1 — Understand the `uptime` Output Structure### Step 2 — Extract Each Value with Multiple Methods### Step 3 — The Complete SolutionBoth files saved correctly. Now the deep explanation.

---

## Part 3: Every Concept Explained Simply

### The `uptime` Output — Every Field Mapped

```
 14:35:22 up 2 days, 5:37,  3 users,  load average: 0.12, 0.45, 0.38
 ├──────┘ ├──────────────┘  ├──────┘  ├────────────┘├───┘ ├───┘ ├───┘
 │         │                 │           │             │     │     │
 time      how long up       who's on    label        1min  5min  15min
```

### `/proc/uptime` vs `uptime` Command

```
cat /proc/uptime
47.12  26.72
  ↑       ↑
  │       └── Total seconds all CPUs spent IDLE since boot
  └────────── Seconds since system boot (wall clock)

Advantages of /proc/uptime over uptime command:
  • Machine-readable (no parsing of human text)
  • Never changes format regardless of OS version
  • Raw seconds = precise, no ambiguity
  • Available even in restricted environments
  • Faster (no process spawn needed with cat)

Convert seconds to human format:
  SECS=172617   (2 days, 3:56:57)
  DAYS=$((172617 / 86400))    = 1   ← wait, 172617/86400 = 1.999... = 1 day
  Actually: 172617 / 86400 = 1 day (int division)
  Let me recalc: 2*86400 = 172800 > 172617, so it's 1 day
  HOURS=$(( (172617 % 86400) / 3600 ))  = (172617 - 86400) / 3600 = 86217/3600 = 23
  MINS=$(( (172617 % 3600) / 60 ))      = (172617 % 3600) / 60 = ...

  Easier: let the OS do it with /proc/uptime integers
```

### `/proc/loadavg` — The Cleanest Source

```
cat /proc/loadavg
0.12  0.45  0.38  2/81  527
  ↑     ↑     ↑    ↑     ↑
  │     │     │    │     └── Last PID created
  │     │     │    └──────── Running processes / Total processes
  │     │     └───────────── 15-min load average ← FIELD 3
  │     └─────────────────── 5-min load average
  └───────────────────────── 1-min load average

Extract 15-min:
  awk '{print $3}' /proc/loadavg   → 0.38
  cut -d' ' -f3 /proc/loadavg      → 0.38

Why this beats parsing uptime:
  No format variations
  No stripping commas
  Always field 3, always a decimal number
```

### Understanding Load Average Numbers

```
Single CPU system:
  Load 0.5 = half the time CPU is waiting
  Load 1.0 = fully utilized (at capacity)
  Load 2.0 = overloaded — tasks waiting

4-core system:
  Load 4.0 = fully utilized (normal)
  Load 8.0 = overloaded

The 15-minute average for auditing:
  Short spike → 1-min goes up, 15-min barely moves
  Sustained load → all three averages elevated
  15-min high + 1-min low = was overloaded, now recovering
  15-min low + 1-min high = just started spiking
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How do you extract the 15-minute load average from `uptime`?"

**Answer:**
> *"Two clean approaches. First and most reliable: read directly from `/proc/loadavg` where field 3 is always the 15-minute average:"*

```bash
awk '{print $3}' /proc/loadavg
# → 0.38

# Or from uptime output — $NF (last field) is always 15-min load:
uptime | awk '{print $NF}'
# → 0.38
```

> *"I prefer `/proc/loadavg` for scripts because it never changes format regardless of system locale, timezone, or uptime duration — it's always three space-separated decimal numbers."*

---

### Q2. "What does load average actually measure?"

**Answer:**
> *"Load average is the average number of processes in a runnable or uninterruptible state over the measured time window. 'Runnable' means actively using CPU or waiting for CPU time. 'Uninterruptible' means waiting for disk I/O or network. A load of 1.0 on a single-core system means the CPU is exactly saturated. On a 4-core system, 4.0 is saturation. The 15-minute average is most useful for auditing because it smooths out short spikes — if 15-minute load is high, the server has been consistently under pressure, not just hit a brief burst."*

---

### Q3. "What's the difference between `uptime` output and `/proc/uptime`?"

**Answer:**
> *"`uptime` is a human-readable command that formats its output for terminal display — the duration format changes based on how long the system has been running (minutes, hours, days). Parsing it requires regex to handle format variations. `/proc/uptime` is a virtual file from the kernel's proc filesystem containing two raw decimal numbers: seconds since boot and total CPU idle time. It never changes format, is always machine-readable, and is available in any environment. For scripting, `/proc/uptime` is more reliable; for a quick human check, `uptime` is more convenient."*

---

### Q4. "How would you alert if 15-minute load exceeds a threshold?"

**Answer:**
```bash
#!/bin/bash
THRESHOLD=2.0
CPU_CORES=$(nproc)
LOAD15=$(awk '{print $3}' /proc/loadavg)

# Compare as floating point using awk
OVERLOADED=$(awk -v load="$LOAD15" -v cores="$CPU_CORES" -v thresh="$THRESHOLD" \
    'BEGIN{ if (load/cores > thresh) print "yes"; else print "no" }')

if [ "$OVERLOADED" = "yes" ]; then
    LOAD_PER_CORE=$(awk -v l="$LOAD15" -v c="$CPU_CORES" 'BEGIN{printf "%.2f", l/c}')
    echo "ALERT: Load ${LOAD15} on ${CPU_CORES} cores (${LOAD_PER_CORE}x capacity)"
    # Send to monitoring: logger, slack webhook, PagerDuty, etc.
fi
```

---

### Q5. "How does load average differ from CPU percentage?"

**Answer:**
> *"CPU percentage measures utilization of a single resource — `top` shows `%CPU` as how much of a CPU core a process is using. Load average counts the queue length — how many processes are waiting for work to complete, including disk I/O wait. A server can have low CPU% but high load average if processes are waiting on slow disk or network. Conversely, a server running a CPU-intensive single process might show 100% CPU but load of 1.0. Load average is more holistic — it captures both CPU and I/O bottlenecks, which is why it's the standard metric for server health audits."*

---

### Q6. "How would you convert `/proc/uptime` seconds to a human-readable string?"

**Answer:**
```bash
TOTAL_SECS=$(awk '{print int($1)}' /proc/uptime)

DAYS=$(( TOTAL_SECS / 86400 ))
HOURS=$(( (TOTAL_SECS % 86400) / 3600 ))
MINS=$(( (TOTAL_SECS % 3600) / 60 ))
SECS=$(( TOTAL_SECS % 60 ))

if [ $DAYS -gt 0 ]; then
    echo "${DAYS} days, ${HOURS}:$(printf '%02d' $MINS)"
elif [ $HOURS -gt 0 ]; then
    echo "${HOURS}:$(printf '%02d' $MINS)"
else
    echo "${MINS} min"
fi

# Or one-liner with awk:
awk '{
    s=int($1); d=s/86400; h=(s%86400)/3600; m=(s%3600)/60
    if(d>0) printf "%d days, %d:%02d\n",d,h,m
    else if(h>0) printf "%d:%02d\n",h,m
    else printf "%d min\n",m
}' /proc/uptime
```

---

## Part 5: Cheat Sheet

```
DATA SOURCES:
  uptime              → human-readable, format varies
  /proc/uptime        → raw: "47.12 26.72" (boot_seconds idle_seconds)
  /proc/loadavg       → raw: "0.12 0.45 0.38 2/81 527"

EXTRACT UPTIME DURATION:
  # From /proc/uptime (most robust):
  awk '{s=int($1);d=s/86400;h=(s%86400)/3600;m=(s%3600)/60;
    if(d>0)printf "%d days, %d:%02d\n",d,h,m;
    else if(h>0)printf "%d:%02d\n",h,m;
    else printf "%d min\n",m}' /proc/uptime

  # From uptime command (simpler but format-dependent):
  uptime | grep -oP '(?<=up )[\d a-z,:]+(?=,\s+\d+ user)'

EXTRACT LOAD AVERAGES:
  awk '{print $1}' /proc/loadavg   # 1-min
  awk '{print $2}' /proc/loadavg   # 5-min
  awk '{print $3}' /proc/loadavg   # 15-min ← audit metric
  uptime | awk '{print $NF}'       # 15-min from uptime

SAVE TO FILES:
  awk '{s=int($1);...}' /proc/uptime > /home/devops/uptime.txt
  awk '{print $3}' /proc/loadavg > /home/devops/loadavg.txt

LOAD INTERPRETATION:
  load / nproc < 1.0  → healthy
  load / nproc = 1.0  → at capacity
  load / nproc > 1.0  → overloaded

LOAD THRESHOLD CHECK (bash can't do floats):
  awk -v l=$(awk '{print $3}' /proc/loadavg) \
      -v c=$(nproc) \
      'BEGIN{if(l/c>1.5) print "OVERLOADED"; else print "OK"}'

PROC FILES:
  /proc/uptime    → Field 1: seconds since boot
  /proc/loadavg   → Field 3: 15-min load average
  /proc/cpuinfo   → processor count (or use nproc)
```

> **Microsoft interview tip:** Azure Monitor and Windows Admin Center both surface Linux load averages for hybrid server management — mention that in production you'd ship these metrics to Azure Monitor with a custom Log Analytics workspace query, or expose them via `/metrics` for Prometheus scraping. Knowing the threshold formula (`load / CPU cores > 1.0`) and explaining the difference between load average and CPU% shows depth that junior candidates miss. That distinction — I/O wait contributing to load but not CPU% — is exactly the kind of nuance Microsoft's SRE teams deal with on Azure Linux VMs.

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
# Using Unmounted Partitions
> **Company:** RedHat | **Difficulty:** Medium
---
#### **Scenario**
The server has unmounted partitions that are not being used and could be utilized for additional storage.
#### **Task**
Identify unmounted partitions that are safe to use (avoiding system-critical partitions like `/`, `/boot`, `/boot/efi`, or swap), create an ext4 filesystem on one with a label `data_extra`, mount it at `/mnt/test`, and verify it's accessible.
#### **Example**
```
# Before (unmounted partition unused)
Block devices scanned, unmounted partitions found
loop0p2: 20GB unmounted, no filesystem
```
```
# After (partition formatted and mounted)
Filesystem created: ext4 with label=data_extra on /dev/loop0p2
Mounted at: /mnt/test
/dev/loop0p2 on /mnt/test type ext4 (rw,relatime)
Partition ready for use
```
---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/using-unmounted-partitions)


## Using Unmounted Partitions — Full Deep Dive

---

## Part 1: Understand It Simply

### What Is a Partition?

```
Physical disk (e.g., /dev/sdb — 100GB):
  ├── /dev/sdb1  (20GB) ← partition 1 — maybe mounted at /home
  ├── /dev/sdb2  (50GB) ← partition 2 — UNMOUNTED, unused ← our target
  └── /dev/sdb3  (30GB) ← partition 3 — swap

Block device types in Linux:
  /dev/sda, /dev/sdb  → SATA/SAS drives
  /dev/nvme0n1        → NVMe drives
  /dev/vda, /dev/vdb  → Virtual drives (KVM/QEMU)
  /dev/xvda           → Xen virtual drives (AWS EC2)
  /dev/loop0          → Loop devices (files used as block devices)
```

### The Three-Step Process

```
Step 1: IDENTIFY
  lsblk    → see all block devices and mount points
  blkid    → see which have filesystems already
  → find one with no filesystem AND not mounted

Step 2: FORMAT
  mkfs.ext4 -L data_extra /dev/sdb2
  → writes ext4 filesystem metadata to the partition
  → labels it "data_extra" for easy identification

Step 3: MOUNT
  mkdir -p /mnt/test
  mount /dev/sdb2 /mnt/test
  → attaches the filesystem to the directory tree
  → /mnt/test is now the "door" into that partition
```

---

## Part 2: Build and Execute

### Step 1 — Create a Realistic Unmounted Partition (Loop Device)### Step 2 — The Full Investigation: Find Safe Partitions### Step 3 — Safety Check: Avoid Critical Partitions### Step 4 — Create Filesystem + Mount + VerifyAll steps complete. `/dev/loop0 on /mnt/test type ext4 (rw,relatime)` — exactly matching the expected output.

---

## Part 3: Every Concept Explained Simply

### Block Devices vs Filesystems vs Mount Points

```
Three separate layers:

LAYER 1: Block Device (hardware level)
  /dev/sdb2    ← just a raw sequence of bytes on a disk
  No structure, no files, just sectors

LAYER 2: Filesystem (structure layer)
  mkfs.ext4 /dev/sdb2   ← writes inode tables, journal, superblock
  Now the raw bytes have structure: directories, files, metadata

LAYER 3: Mount Point (access layer)
  mount /dev/sdb2 /mnt/test   ← attaches filesystem to directory tree
  /mnt/test is now the "door" into that filesystem

Without Step 2: mount fails ("wrong fs type" error)
Without Step 3: filesystem exists but is inaccessible
```

### `lsblk` — The Best Tool for Discovery

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,LABEL

NAME     SIZE  TYPE  FSTYPE  MOUNTPOINT  LABEL
sda      500G  disk
├─sda1   512M  part  vfat    /boot/efi   EFI
├─sda2     1G  part  ext4    /boot
├─sda3   498G  part  ext4    /
sdb      100G  disk
├─sdb1    20G  part  ext4    /home       home_data
└─sdb2    80G  part          ←←← NO FSTYPE, NO MOUNTPOINT = our target
                               ^^ BLANK = no filesystem yet
                               ^^ BLANK = not mounted = SAFE TO USE
```

### `mkfs.ext4` — What It Actually Does

```
mkfs.ext4 -L data_extra /dev/loop0

mkfs.ext4  = make filesystem, ext4 type
-L         = assign a LABEL (human-readable name)
             → findmnt shows it, blkid shows it
             → can mount by label: mount LABEL=data_extra /mnt/test
             → useful when /dev/sdX names change across reboots

What mkfs.ext4 writes to the device:
  Superblock       ← filesystem metadata (size, block count, UUID)
  Inode table      ← one inode per possible file
  Block groups     ← where data blocks live
  Journal          ← ext4 journaling for crash recovery
  lost+found/      ← recovery directory (created automatically)

After mkfs.ext4:
  blkid shows: TYPE="ext4" LABEL="data_extra" UUID="..."
  The UUID never changes even if the device path does ← production gold
```

### The `mount` Command — What Options Mean

```
mount /dev/loop0 /mnt/test
# Uses defaults: rw, relatime, errors=remount-ro

After mounting, mount shows:
  /dev/loop0 on /mnt/test type ext4 (rw,relatime)
                                     ↑↑  ↑
                                     │   └── relatime: atime updated
                                     │        only when file modified
                                     └── rw: read-write (not read-only)

Common mount options:
  ro          → read-only (for snapshots, forensics)
  noexec      → prevent executing binaries from this filesystem
  nosuid      → ignore setuid bits (security hardening)
  noatime     → don't update access times (performance)
  defaults    → rw,suid,dev,exec,auto,nouser,async
```

### Making the Mount Permanent — `/etc/fstab`

```
/etc/fstab = filesystem table — mounts applied at boot time

Without fstab entry:
  Mount is TEMPORARY — lost after reboot

Add to /etc/fstab:
  UUID=e6e66c4a-1f9f-400f-8d8c-ed0de1089044  /mnt/test  ext4  defaults  0  2

  Field 1: UUID=...    ← identify by UUID (never by /dev/sdX which can change!)
  Field 2: /mnt/test   ← where to mount
  Field 3: ext4        ← filesystem type
  Field 4: defaults    ← mount options
  Field 5: 0           ← dump (backup): 0=skip
  Field 6: 2           ← fsck order: 0=skip, 1=root first, 2=other

Test without rebooting:
  mount -a    ← mounts everything in /etc/fstab not yet mounted
```

---

## Part 4: Interview Questions — Detailed Answers

---

### Q1. "How do you find unmounted partitions that are safe to use?"

**Answer:**
```bash
# lsblk shows everything at a glance:
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT

# Unmounted = MOUNTPOINT is empty
# Safe = not swap, not /, not /boot

# Programmatic approach:
lsblk -J | python3 -c "
import json,sys
data = json.load(sys.stdin)
def check(dev):
    name = dev.get('name','')
    mnt = dev.get('mountpoint') or ''
    fstype = dev.get('fstype') or ''
    critical = ['/', '/boot', '/boot/efi', '[SWAP]']
    if not mnt and dev.get('type') in ['part','loop']:
        print(f'  CANDIDATE: /dev/{name} ({dev[\"size\"]}) fstype={fstype or \"<none>\"}')
    for child in dev.get('children',[]):
        check(child)
for d in data.get('blockdevices',[]): check(d)
"
```

> *"I use `lsblk` because it shows the full hierarchy of disks, partitions, and their current mount state in one view. I look for `part` type entries with an empty MOUNTPOINT and no filesystem, cross-checking against `blkid` to confirm no existing data."*

---

### Q2. "What does `mkfs.ext4 -L data_extra` do? Why use a label?"

**Answer:**
> *"`mkfs.ext4` writes an ext4 filesystem onto the raw block device — it creates the superblock, inode table, block groups, and journal. Without it, the device is just raw bytes with no structure. The `-L data_extra` flag assigns a human-readable label stored in the filesystem's superblock. Labels matter in production because:"*

```bash
# Block device names (sdX) change between reboots or hardware changes:
#   Today:  /dev/sdb2 = your data drive
#   Tomorrow: /dev/sdc2 after adding another disk ← WRONG mount!

# Labels and UUIDs never change:
mount LABEL=data_extra /mnt/test    ← always finds the right device
mount UUID=e6e66c4a-...  /mnt/test  ← even more reliable

# In /etc/fstab always use UUID:
UUID=e6e66c4a-1f9f-400f-8d8c-ed0de1089044  /mnt/test  ext4  defaults  0  2
```

---

### Q3. "What's the difference between `ext4`, `xfs`, and `btrfs`? When would you use each?"

**Answer:**

| Filesystem | Best For | Key Feature |
|------------|----------|-------------|
| `ext4` | General purpose, databases, VMs | Mature, stable, fast recovery |
| `xfs` | Large files, high throughput, RHEL default | Excellent parallel I/O, online grow |
| `btrfs` | Snapshots, RAID, deduplication | CoW snapshots, subvolumes |
| `tmpfs` | RAM-backed temp storage | In-memory, lost on reboot |

> *"For this task, `ext4` is the right choice — it's the most widely supported, works everywhere, and is RedHat's recommended filesystem for general storage. XFS is RHEL's default for the root filesystem, optimized for large files. Btrfs adds snapshot capabilities but has more operational complexity."*

---

### Q4. "How do you make the mount survive a reboot?"

**Answer:**
```bash
# Get the UUID (never use /dev/sdX in fstab):
UUID=$(blkid -o value -s UUID /dev/loop0)
echo "UUID=$UUID  /mnt/test  ext4  defaults  0  2" >> /etc/fstab

# Test the fstab entry without rebooting:
mount -a     # mounts all fstab entries not yet mounted
# OR:
umount /mnt/test && mount /mnt/test  # unmount and remount from fstab

# Validate fstab syntax before reboot:
findmnt --verify --fstab
# A syntax error in fstab can prevent the server from booting!
```

---

### Q5. "How do you safely confirm a partition has no data before formatting?"

**Answer:**
```bash
# Check 1: no filesystem (would be overwritten)
blkid /dev/sdb2
# If blank output → no filesystem ✓
# If shows ext4/xfs/etc → DATA EXISTS — confirm with owner before proceeding

# Check 2: not mounted
mount | grep "/dev/sdb2"
grep "/dev/sdb2" /proc/mounts
# Should return nothing ✓

# Check 3: not swap
swapon --show | grep "/dev/sdb2"
# Should return nothing ✓

# Check 4: not in /etc/fstab (already configured for something)
grep "sdb2\|$(blkid -o value -s UUID /dev/sdb2)" /etc/fstab
# Should return nothing ✓

# Check 5: hexdump to confirm it's truly empty
hexdump -C /dev/sdb2 | head -5
# All zeros = blank ✓
# Non-zero data = something was written here
```

---

### Q6. "What is a loop device and why did we use one here?"

**Answer:**
> *"A loop device makes a regular file act like a block device — it 'loops' file I/O through a virtual device `/dev/loopN`. `losetup -f --show file.img` attaches the file as the next available loop device. This is how Linux mounts ISO images, container layers, disk images, and how `mkfs` and `mount` can work on files. In production you'd use a real partition (`/dev/sdb1`), but loop devices are identical from the kernel's perspective — `mkfs.ext4`, `mount`, and `blkid` work identically on both. They're widely used in containers, cloud init disks, and testing."*

---

## Part 5: Cheat Sheet

```
DISCOVER UNMOUNTED PARTITIONS:
  lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,LABEL
  blkid                              # show existing filesystem types
  lsblk -J                           # JSON output for scripting

SAFETY CHECKS (avoid these):
  grep "/ \|/boot" /proc/mounts      # critical mounted filesystems
  swapon --show                      # active swap
  grep -E "^/|^UUID" /etc/fstab      # fstab entries

FORMAT WITH LABEL:
  mkfs.ext4 -L data_extra /dev/sdb2  # ext4 with label
  mkfs.xfs  -L data_extra /dev/sdb2  # xfs with label
  mkfs.ext4 -n /dev/sdb2             # DRY RUN (no write)

MOUNT:
  mkdir -p /mnt/test
  mount /dev/sdb2 /mnt/test
  mount LABEL=data_extra /mnt/test   # by label
  mount UUID=xxxx /mnt/test          # by UUID (most reliable)

VERIFY:
  mount | grep /mnt/test
  findmnt /mnt/test
  df -h /mnt/test
  lsblk -o NAME,FSTYPE,LABEL,MOUNTPOINT

MAKE PERMANENT (/etc/fstab):
  UUID=$(blkid -o value -s UUID /dev/sdb2)
  echo "UUID=$UUID /mnt/test ext4 defaults 0 2" >> /etc/fstab
  mount -a        # apply without reboot
  findmnt --verify --fstab  # validate syntax

UNMOUNT SAFELY:
  umount /mnt/test
  umount /dev/sdb2  # same result, by device

LOOP DEVICE (for testing):
  dd if=/dev/zero of=/tmp/disk.img bs=1M count=200
  losetup -f --show /tmp/disk.img    # attach → /dev/loop0
  losetup -d /dev/loop0              # detach when done
```

> **RedHat interview tip:** RHEL is the enterprise Linux standard — they'll push you on persistence (`/etc/fstab` with UUID), filesystem choice (why ext4 vs xfs for different workloads), and disaster recovery (`e2fsck /dev/sdb2` to check and repair ext4). Mention `tune2fs -l /dev/sdb2` to inspect filesystem metadata without mounting, and `resize2fs` for online partition expansion — that's the operational depth RHEL Certified Engineers are expected to have.
