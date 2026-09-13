# Dracut Emergency Mode: Stale LVM Swap Reference

## Why the Dracut Emergency Mode Happened

Dracut is a small temporary environment used during the early Linux boot process. It helps find storage devices, activate required LVM volumes, prepare the root filesystem, and hand control to Rocky Linux.

```
Power On
   |
   v
GRUB
   |
   v
Kernel + initramfs (Dracut)
   |
   v
Find required LVM volumes
   |
   v
Mount root filesystem
   |
   v
Rocky Linux
```

### The Missing LVM Volume

The old swap Logical Volume was removed:

```
rlm/swap
```

But the boot configuration still contained:

```
rd.lvm.lv=rlm/swap
```

Dracut therefore tried to find an LVM volume that no longer existed. After waiting for the missing volume, the boot process failed and entered the emergency shell.

```
Boot configuration
        |
        v
Looking for rlm/swap
        |
        X
rlm/swap does not exist
        |
        v
Dracut cannot satisfy the requirement
        |
        v
Emergency shell
```

This was a stale boot-time LVM reference, not necessarily filesystem corruption.

## What Was Done to Fix It

### Temporary GRUB Fix

At the GRUB boot menu, press:

```
e
```

Remove:

```
rd.lvm.lv=rlm/swap
```

Leave the root LV reference intact, for example:

```
rd.lvm.lv=rlm/root
```

This tells Dracut to stop looking for the deleted swap LV while still finding the operating system's root LV.

**Why the server booted:**

```
rd.lvm.lv=rlm/root  --> EXISTS  --> Continue boot
rd.lvm.lv=rlm/swap  --> DELETED --> Removed temporarily
```

The root LV still existed, so Dracut could activate it and continue booting.

## What Does `rd.lvm.lv` Mean?

Example:

```
rd.lvm.lv=rlm/swap
```

In practical terms, it tells the early boot environment to activate/use a specific LVM Logical Volume.

| Part | Meaning |
|---|---|
| `rd` | early initramfs / RAM-disk boot environment |
| `lvm` | Logical Volume Manager |
| `lv` | Logical Volume |
| `rlm/swap` | Volume Group `rlm` + Logical Volume `swap` |

So `rd.lvm.lv=rlm/swap` means the early boot environment should activate/use the swap LV in the `rlm` Volume Group.

## Permanent Fix

The GRUB edit is temporary. To remove the stale parameter permanently:

```bash
grubby --update-kernel=ALL --remove-args="rd.lvm.lv=rlm/swap"
```

**Command breakdown:**

| Part | Meaning |
|---|---|
| `grubby` | A tool commonly used on Rocky Linux and other Enterprise Linux systems to manage kernel boot parameters. |
| `--update-kernel=ALL` | Updates all installed kernel boot entries rather than only one kernel. |
| `--remove-args` | Removes the specified kernel parameter: `rd.lvm.lv=rlm/swap` |

This makes the manual GRUB fix persistent for that stale reference.

## Verify the Permanent Fix

Run:

```bash
grubby --info=DEFAULT
```

Inspect the `args=` line. The stale parameter should not be present:

```
rd.lvm.lv=rlm/swap
```

The required root parameter should remain when your system needs it:

```
rd.lvm.lv=rlm/root
```

Check all installed kernels:

```bash
grubby --info=ALL | grep 'rd.lvm.lv'
```

Also verify the LVM layout:

```bash
lvs
lsblk
```

## Before Rebooting

Run:

```bash
grubby --info=DEFAULT
grubby --info=ALL | grep 'rd.lvm.lv'
```

Confirm that `rd.lvm.lv=rlm/swap` is absent and that the root LV still exists:

```bash
lvs
```

Use a planned maintenance window for a production reboot.

## Complete Fix Sequence

1. **Check LVM**
   ```bash
   lvs
   lsblk
   ```

2. **Confirm the old swap LV is gone**
   ```bash
   lvs | grep swap
   ```
   No `rlm/swap` LV should exist if it was intentionally deleted.

3. **Remove the stale boot parameter**
   ```bash
   grubby --update-kernel=ALL --remove-args="rd.lvm.lv=rlm/swap"
   ```

4. **Verify the default kernel**
   ```bash
   grubby --info=DEFAULT
   ```

5. **Check all kernels**
   ```bash
   grubby --info=ALL | grep 'rd.lvm.lv'
   ```

6. **Confirm the root LV**
   ```bash
   lvs
   ```

7. **Reboot during a planned maintenance window**
   ```bash
   reboot
   ```

After reboot:

```bash
findmnt /
lvs
```

## Temporary Fix vs Permanent Fix

| Action | Purpose | Survives Reboot? |
|---|---|---|
| Press `e` in GRUB and remove `rd.lvm.lv=rlm/swap` | Temporary boot fix | No |
| `grubby --update-kernel=ALL --remove-args="rd.lvm.lv=rlm/swap"` | Permanently remove stale parameter | Yes |
| Delete `rlm/swap` LV | Remove the actual LV | Yes |
| `swapoff` | Disable active swap | Usually no, unless persistent configuration is also changed |

## Important Safety Notes

### Do Not Remove the Root Parameter Accidentally

Be careful not to remove:

```
rd.lvm.lv=rlm/root
```

if your system requires it. The root LV contains the operating system.

### Do Not Assume Every Dracut Emergency Is Caused by Swap

Dracut emergency mode can also be caused by:

- Missing root LV
- Incorrect LVM configuration
- Incorrect `/etc/fstab`
- Missing storage device
- Incorrect UUID
- Filesystem problems
- Encryption/unlock problems
- Initramfs problems
- Storage or disk failures

Always inspect the actual error messages before changing boot parameters.

## Relationship to the Earlier Swap Change

```
Original system
      |
      v
rlm/swap existed
      |
      v
swapoff rlm/swap
      |
      v
Remove swap configuration
      |
      v
Delete / repurpose swap LV
      |
      v
rlm/swap no longer exists
      |
      X
Boot configuration still references rlm/swap
      |
      v
Dracut emergency mode
      |
      v
Remove rd.lvm.lv=rlm/swap temporarily
      |
      v
Boot successfully
      |
      v
Make removal permanent with grubby
```

## Golden Rule

When an LVM Logical Volume is deleted, check for references to that LV in:

- `/etc/fstab`
- GRUB/kernel boot parameters
- other system configuration files

For the specific stale boot parameter discussed here:

```
rd.lvm.lv=rlm/swap
```

the permanent removal command is:

```bash
grubby --update-kernel=ALL --remove-args="rd.lvm.lv=rlm/swap"
```

Then verify:

```bash
grubby --info=ALL | grep 'rd.lvm.lv'
```

The goal is:

```
rlm/swap  -> no longer referenced
rlm/root  -> still referenced when required
```

This prevents the same stale `rlm/swap` boot reference from causing the same Dracut failure on subsequent boots.
