
# Fixing Dracut Emergency Mode: Stale LVM Swap Reference

> **Scenario:** A Rocky Linux server enters Dracut emergency mode after a swap Logical Volume (`rlm/swap`) is deleted, because the boot configuration still references it.

---


## Example: GRUB Boot-Parameter Editing Screen

The GRUB boot-entry editor is used for the temporary boot fix. The kernel line contains the required root LVM parameter:

```
rd.lvm.lv=rlm/root
```

When the old swap LV had been deleted, the stale parameter:

```
rd.lvm.lv=rlm/swap
```

needed to be **removed temporarily** from this boot entry.

The screen also shows:

```
loglevel=3
```

which reduces the amount of kernel information printed to the console during boot.

> [!IMPORTANT]
> Editing the GRUB entry with `e` is a **temporary change for that boot only**. A persistent change must be made from the running system using the appropriate boot configuration tool.

---

## Why the Dracut Emergency Mode Happened

### Overview

The server entered Dracut emergency mode because the system was still trying to find an LVM Logical Volume that had already been deleted.

| Item | Value |
|------|-------|
| **Deleted volume** | `rlm/swap` |
| **Stale boot reference** | `rd.lvm.lv=rlm/swap` |

This created a **mismatch** between the actual storage layout and the boot configuration.

---

### 1. The Root Cause: The "Ghost" Drive

Earlier, the old swap Logical Volume was deleted:

```
rlm/swap
```

However, the boot configuration still contained instructions telling the early boot process to look for it.

**Actual storage:**

| LV | Status |
|----|--------|
| `rlm/root` | ✅ EXISTS |
| `rlm/swap` | ❌ DELETED |

**Boot configuration:**

- Look for `rlm/root`
- Look for `rlm/swap`

**The problem:**

```
Boot configuration
       │
       ▼
Looking for rlm/swap
       │
       ✗
rlm/swap no longer exists
```

The old reference became a **stale LVM reference**.

---

### 2. The Crash: Dracut Timeout

During early boot, Rocky Linux starts a temporary environment called **Dracut**. Its job includes:

- Finding required storage
- Activating required LVM volumes
- Preparing the root filesystem
- Starting the normal operating system

**Simplified boot process:**

```
Server Power On
      │
      ▼
     GRUB
      │
      ▼
Kernel + Dracut
      │
      ▼
Find required LVM volumes
      │
      ▼
Find root filesystem
      │
      ▼
Start Rocky Linux
```

Because the old configuration still contained:

```
rd.lvm.lv=rlm/swap
```

Dracut attempted to find `rlm/swap`, but the LV had already been deleted. It waited for the requested storage and eventually **failed**.

The system then entered the emergency shell:

```
sh-5.2#
```

This is why you saw the Dracut emergency mode instead of the normal login screen.

---

### 3. The Temporary Bypass

At the GRUB boot menu, press **`e`** to edit the boot entry.

**Remove:**

```
rd.lvm.lv=rlm/swap
```

and the old:

```
resume=UUID=...
```

reference.

> [!NOTE]
> This is a **temporary fix for that boot only**. The boot entry then stops requesting the deleted swap LV.

The important root LV entry is **left in place**:

```
rd.lvm.lv=rlm/root
```

So the boot process can still find the actual operating-system volume.

---

### 4. Why the Server Booted After the Temporary Fix

**Before the change:**

```
GRUB
 │
 ├── rlm/root  → EXISTS
 │
 ├── rlm/swap  → DELETED
 │              │
 │              ▼
 │          Dracut failure
```

**After editing the GRUB entry:**

```
GRUB
 │
 ├── rlm/root  → EXISTS
 │
 ├── rlm/swap  → REMOVED FROM THIS BOOT
 │              │
 │              ▼
 │        Continue boot
```

Because the root LV still existed, the operating system could start normally.

---

### 5. What Does `rd.lvm.lv` Mean?

**Example:**

```
rd.lvm.lv=rlm/swap
```

This parameter tells the early boot environment to activate/use a specific LVM Logical Volume.

| Component | Meaning |
|-----------|---------|
| `rd` | Early RAM-disk / initramfs environment |
| `lvm` | Logical Volume Manager |
| `lv` | Logical Volume |
| `rlm` | Volume Group name |
| `swap` | Logical Volume name |

> During early boot, activate/use the **swap** Logical Volume inside the **rlm** Volume Group.

---

### 6. What Does `resume=UUID` Do?

The kernel parameter:

```
resume=UUID=...
```

is normally associated with the device used for **hibernation/resume**.

If the device referenced by an old `resume=UUID` parameter has been deleted or is no longer appropriate, it can also become a **stale boot reference**.

In the situation described here, the old `resume=UUID` reference was removed along with the stale swap reference.

---

## The Permanent Fix

> [!WARNING]
> The temporary GRUB edit only applied to that specific boot. A permanent fix is required.

### 7. Remove the Stale Boot Parameter

```bash
grubby --update-kernel=ALL --remove-args="rd.lvm.lv=rlm/swap"
```

This removes `rd.lvm.lv=rlm/swap` from the configured kernel boot arguments.

---

### 8. Command Breakdown

| Flag | Description |
|------|-------------|
| `grubby` | Tool used on Rocky Linux / Enterprise Linux to manage kernel boot parameters. Updates boot entries without manually editing generated GRUB configuration files. |
| `--update-kernel=ALL` | Applies the change to **all** installed kernel entries, preventing an old kernel from retaining the stale parameter. |
| `--remove-args="rd.lvm.lv=rlm/swap"` | Removes this specific kernel argument from the boot configuration. |

---

### 9. Rebuild the Initramfs

After changing boot parameters, rebuild the initramfs:

```bash
dracut -f
```

**Flow:**

```
grubby
  │
  ▼
Remove stale boot argument
  │
  ▼
dracut -f
  │
  ▼
Rebuild initramfs
  │
  ▼
Future boot uses updated configuration
```

---

### 10. Verify the Permanent Fix

**Check the default kernel entry:**

```bash
grubby --info=DEFAULT
```

Look at the `args=` line. You should **no longer** see:

```
rd.lvm.lv=rlm/swap
```

**Check all kernels:**

```bash
grubby --info=ALL | grep 'rd.lvm.lv'
```

Make sure the deleted LV is not still referenced.

**Expected result:**

| LV | Status |
|----|--------|
| `rlm/swap` | No stale reference |
| `rlm/root` | Remains where required |

---

### 11. Verify the LVM Layout

Check the current LVM configuration:

```bash
lvs
```

Confirm that `rlm/root` exists. If the old swap LV was intentionally deleted, there should no longer be `rlm/swap`.

Review the complete storage layout:

```bash
lsblk
```

---

### 12. Complete Repair Sequence

| Step | Action | Command |
|------|--------|---------|
| 1 | Check LVM | `lvs` / `lsblk` |
| 2 | Confirm the old swap LV is gone | `lvs \| grep swap` |
| 3 | Remove the stale boot reference | `grubby --update-kernel=ALL --remove-args="rd.lvm.lv=rlm/swap"` |
| 4 | Rebuild initramfs | `dracut -f` |
| 5 | Verify the default kernel | `grubby --info=DEFAULT` |
| 6 | Check all kernels | `grubby --info=ALL \| grep 'rd.lvm.lv'` |
| 7 | Confirm the root LV | `lvs` |
| 8 | Reboot during a planned maintenance window | `reboot` |

---

## Reference Tables

### 13. Temporary Fix vs Permanent Fix

| Action | Purpose | Survives Reboot? |
|--------|---------|:----------------:|
| Press `e` in GRUB and remove `rd.lvm.lv=rlm/swap` | One-time boot bypass | ❌ No |
| Remove `resume=UUID` from the GRUB entry | One-time removal of stale resume reference | ❌ No |
| `grubby --update-kernel=ALL --remove-args="rd.lvm.lv=rlm/swap"` | Permanently remove stale LVM parameter | ✅ Yes |
| `dracut -f` | Rebuild initramfs | ✅ Yes |
| Delete `rlm/swap` | Removes the actual LV | ✅ Yes |

---

## 14. Important Safety Notes

> [!CAUTION]
> **Do not remove the root LV parameter accidentally.** Do not confuse `rd.lvm.lv=rlm/root` with `rd.lvm.lv=rlm/swap`. The root LV contains the operating system.

> [!NOTE]
> **Do not assume every Dracut failure is caused by swap.** Dracut emergency mode can have many causes:
>
> - Missing root LV
> - Incorrect LVM configuration
> - Incorrect `/etc/fstab`
> - Missing disk
> - Incorrect UUID
> - Filesystem problems
> - Encryption/unlock problems
> - Initramfs problems
> - Storage failures
>
> **Always check the actual error messages** before making boot changes.

---

## 15. Relationship to the Earlier Swap Change

**Full incident timeline:**

```
Old system
    │
    ▼
rlm/swap exists
    │
    ▼
swapoff /dev/mapper/rlm-swap
    │
    ▼
Swap disabled
    │
    ▼
rlm/swap deleted / repurposed
    │
    ▼
Boot configuration still references rlm/swap
    │
    ▼
Server reboot
    │
    ▼
Dracut looks for rlm/swap
    │
    ✗
LV does not exist
    │
    ▼
Dracut emergency mode
    │
    ▼
Temporary GRUB edit → Remove: rd.lvm.lv=rlm/swap
    │
    ▼
Server boots
    │
    ▼
Permanent grubby change
    │
    ▼
dracut -f
    │
    ▼
Updated boot configuration ✔
```

---

## 16. Golden Rule

> [!TIP]
> Whenever an LVM Logical Volume is intentionally deleted, **check for references** to that LV in:
>
> - `/etc/fstab`
> - GRUB / kernel boot parameters
> - initramfs configuration
> - resume / hibernation configuration
> - other system configuration

**Permanent removal command:**

```bash
grubby --update-kernel=ALL --remove-args="rd.lvm.lv=rlm/swap"
```

**Verify:**

```bash
grubby --info=ALL | grep 'rd.lvm.lv'
```

**Goal:**

| LV | Expected State |
|----|----------------|
| `rlm/swap` | No longer referenced |
| `rlm/root` | Still available and referenced where required |

