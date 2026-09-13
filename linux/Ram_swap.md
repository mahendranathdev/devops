# Linux Swap Management: When to Check, Disable, Clear, or Keep Swap

## 1. Purpose

This README explains Linux RAM and swap usage, how to interpret `free -h`, and when it is safe or unsafe to disable swap.

The examples are based on these two situations:

**Server A**
```
Mem:   3.5Gi total   1.7Gi used   1.2Gi free   1.8Gi available
Swap:  3.9Gi total   0B used     3.9Gi free
```

**Server B**
```
Mem:   62Gi total    3.0Gi used   57Gi free   2.1Gi buff/cache   59Gi available
Swap:  31Gi total    6.2Gi used   25Gi free
```

## 2. What is Swap?

Swap is disk space that Linux can use as an extension of memory.

Normally:

```
Applications
     |
     v
    RAM
     |
     | RAM pressure
     v
   SWAP
```

RAM is much faster than disk-based swap.

Swap is therefore useful as a safety buffer, but it should not normally be treated as a replacement for sufficient RAM.

## 3. Understanding `free -h`

Run:

```bash
free -h
```

Typical output:

```
               total   used   free   shared   buff/cache   available
Mem:             ...
Swap:            ...
```

### RAM columns

**`total`** — Total physical RAM available to the operating system.
Example: `Mem: 62Gi` → the server has approximately 62 GiB of RAM.

**`used`** — RAM currently being used. Do not judge memory pressure from used alone.

**`free`** — RAM that is completely unused at the moment. Linux normally uses otherwise-unused RAM for caching, so a low free value is not automatically bad.

**`buff/cache`** — RAM used for filesystem and kernel caches. Linux can reclaim much of this memory when applications need it.

**`available`** — One of the most useful values when evaluating whether the server is running out of RAM.
Example: `available = 59Gi` means the system estimates that about 59 GiB can be made available to applications without serious memory pressure.

## 4. Understanding Swap

Example:

```
Swap: 31Gi  6.2Gi  25Gi
```

This means:

- Total swap = 31 GiB
- Used swap = 6.2 GiB
- Free swap = 25 GiB

> **Important:** Swap being used does NOT automatically mean there is a problem.

Linux can move relatively inactive memory pages to swap and use the freed RAM for filesystem cache or other purposes. Therefore this is possible:

```
RAM available = 59 GiB
Swap used     = 6.2 GiB
```

It does not necessarily mean the server is currently short of RAM.

## 5. The Most Important Difference

Compare these two situations.

### Situation A: Healthy

```
RAM available = 59 GiB
Swap used     = 6.2 GiB
```

There is plenty of available RAM. Swap usage alone is not a reason to panic.

### Situation B: Potential memory pressure

```
RAM available = 500 MiB
Swap used     = 30 GiB
```

This is much more concerning, especially if swap activity is continuously increasing. Possible symptoms include:

- Slow applications
- High I/O wait
- System responsiveness problems
- Processes being killed by the OOM killer
- Increasing swap-in/swap-out activity

## 6. How to Check Active Swap

Run:

```bash
swapon --show
```

Example:

```
NAME                  TYPE       SIZE  USED PRIO
/dev/mapper/rlm-swap  partition  3.9G  0B   -2
```

This tells you which swap device is active.

Also run:

```bash
cat /proc/swaps
```

If no swap is active, normally only the header is displayed.

## 7. Check Swap Activity

Run:

```bash
vmstat 1 5
```

Look at:

- `si` = swap-in activity
- `so` = swap-out activity

If they remain:

```
0  0
```

there is no significant ongoing swap movement during the sample.

If `si` and `so` are continuously high, investigate memory pressure before disabling swap.

## 8. Check Swappiness

Run:

```bash
cat /proc/sys/vm/swappiness
```

Swappiness controls how aggressively Linux considers swapping. It is a tuning parameter, not a simple ON/OFF switch.

Do not change it blindly on a production server.

## 9. When `swapoff` Can Be Considered

```bash
swapoff /dev/mapper/rlm-swap
```

This disables that specific swap device. It can be considered when:

- You have confirmed enough available RAM.
- Swap usage is zero or very small.
- You have a clear operational reason to disable swap.
- You understand the server's workload.
- You have checked that disabling swap will not create memory pressure.
- You are prepared to monitor the server afterward.

Example:

```
RAM available = 1.8 GiB
Swap used     = 0 B
```

This was the situation on Server A. Because swap was already unused, `swapoff` did not need to move swapped pages back into RAM.

## 10. When NOT to Run `swapoff`

Do NOT blindly run:

```bash
swapoff -a
```

or:

```bash
swapoff /dev/mapper/rlm-swap
```

when:

- Available RAM is low.
- Swap usage is large and active.
- `vmstat` shows continuous si/so.
- The server is already under memory pressure.
- You do not know what workloads are running.
- It is a critical production system and you have not planned the change.

For example:

```
RAM available = 500 MiB
Swap used     = 20 GiB
```

Do not immediately disable swap. The system may need that swap to avoid an OOM condition.

## 11. What Happens During `swapoff`?

When you run:

```bash
swapoff /dev/mapper/rlm-swap
```

Linux stops using that swap device. If pages are currently stored in swap, Linux must bring them back into RAM.

Therefore:

```
Swap used = 0
```

does not mean the data disappeared. It means the pages are no longer stored in that swap area.

If there is insufficient RAM, `swapoff` can fail or create severe memory pressure.

## 12. Removing Swap Permanently

Disabling swap temporarily:

```bash
swapoff /dev/mapper/rlm-swap
```

does NOT necessarily prevent it from being enabled after reboot.

To prevent automatic activation, the swap entry normally needs to be removed or commented out in `/etc/fstab`.

**Always make a backup first:**

```bash
cp /etc/fstab /etc/fstab.bak
```

Then inspect:

```bash
grep -i swap /etc/fstab
```

Do not delete anything until you confirm the correct line.

## 13. `sed` Example

The command:

```bash
sed -i '/swap/d' /etc/fstab
```

means: delete every line containing the text `swap`.

This worked in the previous example. However, this command can be too broad for production use because it may remove more than the intended swap entry if multiple lines contain the word `swap`.

A safer approach is to inspect first:

```bash
grep -i swap /etc/fstab
```

Then edit the specific line deliberately. After editing, verify:

```bash
grep -i swap /etc/fstab
```

## 14. Why the First `sed` Command Failed

This command:

```bash
sed -i 'swap/d' /etc/fstab
```

failed with:

```
unterminated `s' command
```

The correct pattern syntax is:

```bash
sed -i '/swap/d' /etc/fstab
```

The `/swap/` portion identifies the matching text. The `d` means delete.

General form:

```
/pattern/d
```

## 15. Recommended Investigation Procedure

Before changing swap on a server, use this sequence:

1. **Check RAM and swap** — `free -h`
2. **Identify active swap** — `swapon --show`
3. **Check swap activity** — `vmstat 1 5`
4. **Check swappiness** — `cat /proc/sys/vm/swappiness`
5. **Check persistent configuration** — `grep -i swap /etc/fstab`

Then decide whether a change is actually necessary.

## 16. Decision Guide

| Situation | Recommended action |
|---|---|
| Plenty of available RAM + swap used but no active swap activity | Usually leave it alone |
| Plenty of RAM + swap used = 0 | `swapoff` may be reasonable if there is a specific reason |
| Low available RAM + swap heavily used | Do NOT blindly run `swapoff` |
| High continuous si/so | Investigate memory pressure |
| Production server | Plan and monitor changes |
| Need temporary swap disable | Use `swapoff` |
| Need swap disabled after reboot | Update `/etc/fstab` after verifying the correct entry |
| Unsure whether swap is needed | Keep it until you understand the workload |

## 17. Server A Analysis

**Original:**

```
Mem:   3.5Gi total
       1.7Gi used
       1.2Gi free
       1.8Gi available

Swap:  3.9Gi total
       0B used
       3.9Gi free
```

This showed:

- Swap existed.
- Swap was not being used.
- There was approximately 1.8 GiB of available RAM.

**After:**

```bash
swapoff /dev/mapper/rlm-swap
```

the result became:

```
Swap: 0B  0B  0B
```

RAM remained almost unchanged because there were no pages in swap that needed to be brought back. This was an expected result.

## 18. Server B Analysis

**Output:**

```
Mem:   62Gi total
       3.0Gi used
       57Gi free
       2.1Gi buff/cache
       59Gi available

Swap:  31Gi total
       6.2Gi used
       25Gi free
```

This server has a very large amount of available RAM. The 6.2 GiB swap usage by itself is NOT enough evidence that swap should be disabled.

First check:

```bash
swapon --show
vmstat 1 5
cat /proc/sys/vm/swappiness
```

If swap activity is not ongoing and the server is healthy, there may be no reason to change anything.

## 19. Important Production Rule

Do not use this rule:

> "Swap is being used, therefore I must disable or clear swap."

Use this rule instead:

> "Check available RAM, current swap activity, workload, and the reason for changing swap before taking action."

## 20. Useful Commands

| Purpose | Command |
|---|---|
| RAM and swap summary | `free -h` |
| Active swap devices | `swapon --show` |
| Swap information | `cat /proc/swaps` |
| Swap activity | `vmstat 1 5` |
| Swappiness | `cat /proc/sys/vm/swappiness` |
| Persistent swap configuration | `grep -i swap /etc/fstab` |
| Backup fstab | `cp /etc/fstab /etc/fstab.bak` |
| Disable a specific swap device | `swapoff /dev/mapper/rlm-swap` |
| Disable all active swap | `swapoff -a` |

Use the last command carefully on production systems.

## 21. Quick Cheat Sheet

```
                 CHECK RAM
                    |
                    v
                free -h
                    |
          +---------+---------+
          |                   |
     RAM available       RAM available
        is high              is low
          |                   |
          v                   v
    Check swap activity   Be careful
      vmstat 1 5          with swapoff
          |
          v
    si/so continuously high?
       /          \
     YES           NO
      |             |
 Investigate      Usually
 memory pressure  no urgent action
```

## 22. Final Recommendation

For normal Linux administration:

**Keep swap when:**

- The server is a production system.
- You have enough RAM but want an additional safety buffer.
- There is no specific requirement to remove it.
- Swap usage is not causing performance problems.

**Consider disabling swap when:**

- There is a specific requirement.
- You have confirmed sufficient RAM.
- You have checked swap activity.
- The workload and consequences are understood.
- You can monitor the server after the change.

**Never disable swap blindly when:**

```
available RAM is low
AND
swap usage/activity is high
```

That combination can indicate real memory pressure.

## 23. Golden Rule

> Swap usage is a metric to investigate, not automatically a problem to fix.

Always check:

```bash
free -h
swapon --show
vmstat 1 5
cat /proc/sys/vm/swappiness
grep -i swap /etc/fstab
```

before making a production swap change.
