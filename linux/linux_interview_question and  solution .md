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


