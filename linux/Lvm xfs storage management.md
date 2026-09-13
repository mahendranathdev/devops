# LVM and XFS Storage Management Guide

> **WARNING:** Several commands below are destructive. Always verify the LV and back up important data before running `mkfs` or `lvremove`.

## Overview

This guide covers:

- Converting an old swap LV into `/data`
- Making `/data` permanent with `/etc/fstab`
- Moving 1 GB from data to root
- Recreating the data LV with the remaining space
- Safely testing `/etc/fstab`
- Setting ownership and permissions

## Part 1: Convert the Old Swap LV into a Data Drive

### Before Starting

Check the current configuration:

```bash
free -h
swapon --show
lsblk
vgs
lvs
```

If the old swap LV is active, disable it:

```bash
swapoff /dev/mapper/rlm-swap
```

Verify:

```bash
swapon --show
```

`swapoff` disables swap. It does not erase the old swap signature.

### 1. Rename the Logical Volume

```bash
lvrename rlm swap data
```

**What it does:** Changes `rlm/swap` to `rlm/data`. The device becomes `/dev/mapper/rlm-data`.

**Why:** The LV is no longer being used as swap, so the name `data` accurately describes its new purpose.

**Verify:**

```bash
lvs
```

### 2. Format the Volume as XFS

```bash
mkfs.xfs -f /dev/mapper/rlm-data
```

**What it does:** Creates a new XFS filesystem on the LV.

**Why `-f`?** The old LV may still contain a swap signature. `swapoff` stops swap usage but does not erase that signature. The `-f` option forces XFS formatting.

> **Critical warning:** This destroys existing data on the volume. Before running it, make sure `/dev/mapper/rlm-data` is the correct LV and that its old contents are no longer required.

### 3. Create the Mount Point and Mount the Volume

```bash
mkdir -p /data
mount /dev/mapper/rlm-data /data
```

**What it does:** Creates `/data` and attaches the XFS filesystem to it. After mounting, files written to `/data` are stored on the `rlm-data` LV.

**Verify:**

```bash
df -h /data
findmnt /data
```

### 4. Make the Mount Permanent

First create a backup:

```bash
cp /etc/fstab /etc/fstab.bak
```

Add the mount entry:

```bash
echo "/dev/mapper/rlm-data /data xfs defaults 0 0" >> /etc/fstab
```

**What it does:** Adds this line to `/etc/fstab`:

```
/dev/mapper/rlm-data /data xfs defaults 0 0
```

Linux can then mount `/data` automatically during boot.

**Important: `>` vs `>>`**

Use `>>` because it means append. Do not accidentally use `>` because `>` means overwrite. For example:

```bash
echo "something" > /etc/fstab
```

can overwrite the entire `/etc/fstab` file and cause serious system problems.

**Verify:**

```bash
grep -i '/data' /etc/fstab
```

## Part 2: Move 1 GB from Data to Root

### Important XFS Rule

XFS can grow, but it cannot be shrunk in place. Therefore, if you need to move 1 GB from an XFS data LV to root, the simple approach is:

```
Unmount data
      |
      v
Delete data LV
      |
      v
Give 1 GB to root
      |
      v
Create data again
      |
      v
Format data as XFS
      |
      v
Mount data
```

> **Warning:** This destroys everything stored on `/data`. Back up important files first.

### 1. Unmount `/data`

```bash
umount /data
```

**Verify:**

```bash
findmnt /data
```

The filesystem should no longer be mounted.

### 2. Delete the Data LV

```bash
lvremove -y /dev/mapper/rlm-data
```

**What it does:** Deletes the data Logical Volume and returns its capacity to the free space in the `rlm` Volume Group.

> **Critical warning:** This destroys the filesystem and all data stored on it.

**Verify first:**

```bash
lvs
```

### 3. Add 1 GB to Root

```bash
lvextend -r -L +1G /dev/mapper/rlm-root
```

**What it does:** Adds exactly 1 GB to the root LV.

- `-L +1G` — add 1 GB
- `-r` — resizes the filesystem as well

**Verify:**

```bash
lvs
df -h /
```

### 4. Recreate the Data LV

Use the remaining free space:

```bash
lvcreate -l 100%FREE -n data rlm
```

Create a new XFS filesystem:

```bash
mkfs.xfs /dev/mapper/rlm-data
```

**What it does:** Creates a new data LV from all remaining free space and creates a new empty XFS filesystem. The new `/data` filesystem is empty — old data is not restored automatically.

### 5. Mount the New Data Volume

Because `/etc/fstab` already contains the `/data` entry:

```bash
mount -a
```

**Verify:**

```bash
df -h /data
findmnt /data
```

## Part 3: Safely Verify `/etc/fstab`

### 1. Check the Configuration

```bash
findmnt --verify
```

This checks the structure and validity of the mount information in `/etc/fstab`. If errors or warnings are reported, fix them before rebooting.

### 2. Perform a Dry Run

```bash
mount -fav
```

Meaning:

- `-f` — fake/dry run
- `-a` — process all entries in `/etc/fstab`
- `-v` — verbose

This allows you to inspect what `mount` would do without performing the normal mounts. If something is already mounted, it may be reported as ignored — that can be expected.

### 3. Perform a Live Test

```bash
umount /data
mount -a
```

Then verify:

```bash
df -h /data
findmnt /data
```

If `/data` successfully mounts using `/etc/fstab`, the configuration has passed an important practical test.

No configuration test can literally guarantee that a reboot will succeed. Keep the `/etc/fstab.bak` backup and use a planned maintenance window for production systems.

## Part 4: Set Ownership and Permissions

A newly created filesystem is normally controlled by root.

Check:

```bash
ls -ld /data
```

### 1. Change Ownership

Replace `your_username` with the actual Linux account:

```bash
chown -R your_username:your_username /data
```

Example:

```bash
chown -R mahendra:mahendra /data
```

**Verify:**

```bash
ls -ld /data
```

### 2. Set Permissions

```bash
chmod -R 755 /data
```

**Meaning of `755`:**

```
Owner:  rwx
Group:  r-x
Others: r-x
```

> **Security note:** `chmod -R 755 /data` changes permissions recursively. Do not blindly use recursive 755 for sensitive application data, databases, credentials, or files that require restricted permissions. Set permissions according to the application and security requirements.

### 3. Test User Access

Switch to the intended standard user and run:

```bash
touch /data/testfile
```

Verify:

```bash
ls -l /data/testfile
```

Then remove the test file:

```bash
rm /data/testfile
```

## Complete Command Sequence

### Part 1: Old Swap LV → Data

```bash
# Check
free -h
swapon --show
lsblk
vgs
lvs

# Disable swap
swapoff /dev/mapper/rlm-swap

# Rename
lvrename rlm swap data

# Format
mkfs.xfs -f /dev/mapper/rlm-data

# Create mount point
mkdir -p /data

# Mount
mount /dev/mapper/rlm-data /data

# Back up fstab
cp /etc/fstab /etc/fstab.bak

# Add permanent mount
echo "/dev/mapper/rlm-data /data xfs defaults 0 0" >> /etc/fstab

# Verify
findmnt --verify
mount -fav
df -h /data
findmnt /data
```

### Part 2: Move 1 GB from Data to Root

> **Warning:** This destroys the existing `/data` filesystem and its contents. Back up important data first.

```bash
umount /data

lvremove -y /dev/mapper/rlm-data

lvextend -r -L +1G /dev/mapper/rlm-root

lvcreate -l 100%FREE -n data rlm

mkfs.xfs /dev/mapper/rlm-data

mount -a

# Verify
df -h /
df -h /data
lvs
findmnt /data
```

## Safety Checklist

**Before `mkfs.xfs`:**

```bash
lvs
lsblk
```

Confirm the target is correct. Never run `mkfs.xfs` on a volume containing data you need.

**Before `lvremove`:**

```bash
lvs
findmnt /data
```

Make sure `/data` is unmounted and important data has been backed up.

**Before editing `/etc/fstab`:**

```bash
cp /etc/fstab /etc/fstab.bak
cat /etc/fstab
```

After editing:

```bash
findmnt --verify
mount -fav
```

**Before rebooting:**

```bash
findmnt --verify
mount -fav
df -h
findmnt /data
```

Fix any errors before rebooting.

## When to Use This Procedure

Use it when:

- An old swap LV is intentionally being repurposed.
- The old swap contents are no longer needed.
- You need an XFS filesystem for `/data`.
- You need to redistribute LVM capacity.
- Important data has been backed up.
- You understand the destructive operations involved.

## When NOT to Use This Procedure

Do not use it blindly when:

- The old swap LV is still required.
- `/data` contains important data without a backup.
- You are unsure which LV you are modifying.
- RAM is under pressure and active swap is heavily used.
- The server is a critical production system without a maintenance plan.
- You have not reviewed `/etc/fstab`.
- You do not have a recovery method if storage configuration causes boot problems.

## Key Concepts

| Command | What it does |
|---|---|
| `swapoff /dev/mapper/rlm-swap` | Disables swap usage. Does not erase the swap signature. |
| `mkfs.xfs /dev/mapper/rlm-data` | Creates a new XFS filesystem and destroys existing filesystem data. |
| `lvremove /dev/mapper/rlm-data` | Deletes the LV and returns its capacity to the Volume Group. |
| `lvextend -r -L +1G /dev/mapper/rlm-root` | Adds 1 GB to root and resizes its filesystem. |
| `lvcreate -l 100%FREE -n data rlm` | Creates `data` using all currently free extents in the `rlm` Volume Group. |
| `/etc/fstab` | Contains persistent filesystem mount instructions. |
| `findmnt --verify` | Checks `/etc/fstab` configuration. |
| `mount -fav` | Performs a verbose fake/dry-run of the mounts defined in `/etc/fstab`. |

## Golden Rules

- Always identify the correct LV before destructive commands.
- Back up important data before `mkfs` or `lvremove`.
- XFS cannot be shrunk in place.
- `swapoff` disables swap but does not erase its signature.
- Use `>>` carefully when appending to `/etc/fstab`.
- Never accidentally overwrite `/etc/fstab` with `>`.
- Verify `/etc/fstab` before rebooting.
- Do not blindly use `chmod -R 755` for sensitive data.
- Do not disable active swap when RAM is under pressure without understanding the consequences.
- For production systems, have a backup and recovery plan before changing storage.

## Quick Reference

```bash
# Memory and swap
free -h
swapon --show

# LVM
pvs
vgs
lvs
lsblk

# Disable old swap
swapoff /dev/mapper/rlm-swap

# Rename
lvrename rlm swap data

# Format
mkfs.xfs -f /dev/mapper/rlm-data

# Mount
mkdir -p /data
mount /dev/mapper/rlm-data /data

# Backup fstab
cp /etc/fstab /etc/fstab.bak

# Permanent mount
echo "/dev/mapper/rlm-data /data xfs defaults 0 0" >> /etc/fstab

# Verify fstab
findmnt --verify
mount -fav

# Move 1 GB from data to root
umount /data
lvremove -y /dev/mapper/rlm-data
lvextend -r -L +1G /dev/mapper/rlm-root

# Recreate data
lvcreate -l 100%FREE -n data rlm
mkfs.xfs /dev/mapper/rlm-data
mount -a

# Final verification
df -h
findmnt /data
lvs
```
