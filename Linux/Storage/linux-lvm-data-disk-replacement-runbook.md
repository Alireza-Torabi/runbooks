# Linux LVM Data Disk Replacement Runbook

## Purpose

This document describes the procedure to replace an existing Linux data disk using LVM.

Supported distributions:

- Ubuntu
- Debian
- RHEL
- Rocky Linux
- AlmaLinux
- CentOS
- SLES

Requirements:

- Data on the old disk can be deleted.
- New disk will be initialized as a new LVM storage.
- Mount point remains the same (example: `/data`).

---

# Scenario

Before:

```
/dev/<OLD_DATA_DISK>
 |
 +-- LVM PV
       |
       +-- VG
             |
             +-- LV
                   |
                   /data
```

After:

```
/dev/<NEW_DATA_DISK>
 |
 +-- LVM PV
       |
       +-- VG
             |
             +-- LV
                   |
                   /data
```

---

# 1. Verify Current Storage

Check current disk layout:

```bash
lsblk
df -h
pvs
vgs
lvs
```

Identify the old data disk.

Example:

```
/dev/sdb
 |
 +-- vg0/lv-0
        |
        /data
```

---

# 2. Stop Services Using the Data Mount

Stop applications using `/data`.

Example:

```bash
systemctl stop <service-name>
```

Check open files:

```bash
lsof +D /data
```

The output should be empty.

---

# 3. Update /etc/fstab Before Disk Removal (IMPORTANT)

Before shutting down the server, remove the old disk dependency.

Check:

```bash
cat /etc/fstab
```

Find the old mount entry.

Examples:

```
UUID=<OLD_UUID> /data ext4 defaults 0 2
```

or:

```
/dev/disk/by-id/dm-uuid-LVM-xxxx /data ext4 defaults 0 1
```

Comment or remove the old entry:

```
#UUID=<OLD_UUID> /data ext4 defaults 0 2
```

Reload systemd:

```bash
systemctl daemon-reload
```

This prevents boot delays caused by missing disks.

---

# 4. Unmount Old Data Disk

Unmount:

```bash
umount /data
```

Verify:

```bash
df -h
```

---

# 5. Remove Old LVM Configuration

## Remove Logical Volume

```bash
lvremove /dev/<VG_NAME>/<LV_NAME>
```

Example:

```bash
lvremove /dev/vg0/lv-0
```

---

## Remove Volume Group

```bash
vgremove <VG_NAME>
```

Example:

```bash
vgremove vg0
```

---

## Remove Physical Volume

```bash
pvremove /dev/<OLD_DATA_DISK>
```

Example:

```bash
pvremove /dev/sdb
```

---

# 6. Shutdown and Replace Disk

Shutdown:

```bash
shutdown -h now
```

From the hypervisor or physical server:

1. Remove the old data disk.
2. Add the new disk.
3. Start the system.

---

# 7. Detect New Disk

Check available disks:

```bash
lsblk
```

Example:

```
/dev/sdb  1T
```

Make sure the new disk is empty.

---

# 8. Create New LVM Structure

## Create Physical Volume

```bash
pvcreate /dev/<NEW_DATA_DISK>
```

Example:

```bash
pvcreate /dev/sdb
```

---

## Create Volume Group

```bash
vgcreate <VG_NAME> /dev/<NEW_DATA_DISK>
```

Example:

```bash
vgcreate vg0 /dev/sdb
```

---

## Create Logical Volume

Use all available space:

```bash
lvcreate -l 100%FREE -n <LV_NAME> <VG_NAME>
```

Example:

```bash
lvcreate -l 100%FREE -n lv-0 vg0
```

Verify:

```bash
pvs
vgs
lvs
```

---

# 9. Create Filesystem

Example: ext4

```bash
mkfs.ext4 /dev/<VG_NAME>/<LV_NAME>
```

Example:

```bash
mkfs.ext4 /dev/vg0/lv-0
```

Example: XFS

```bash
mkfs.xfs /dev/vg0/lv-0
```

---

# 10. Mount New Data Disk

Create mount point:

```bash
mkdir -p /data
```

Mount:

```bash
mount /dev/<VG_NAME>/<LV_NAME> /data
```

Verify:

```bash
df -h
```

---

# 11. Configure Permanent Mount

Get filesystem UUID:

```bash
blkid /dev/<VG_NAME>/<LV_NAME>
```

Example:

```
UUID="xxxx-yyyy"
```

Add to `/etc/fstab`:

For ext4:

```
UUID=xxxx-yyyy /data ext4 defaults 0 2
```

For XFS:

```
UUID=xxxx-yyyy /data xfs defaults 0 0
```

Test:

```bash
mount -a
```

No errors should appear.

---

# 12. Start Services

Start previously stopped services:

```bash
systemctl start <service-name>
```

Verify application functionality.

---

# Validation Checklist

Run:

```bash
lsblk
df -h
pvs
vgs
lvs
```

Expected:

```
System Disk
 |
 +-- Root filesystem


New Data Disk
 |
 +-- LVM PV
       |
       +-- VG
             |
             +-- LV
                   |
                   /data
```

---

# Important Notes

## Disk Names

Disk names are not fixed.

Examples:

```
/dev/sdb
/dev/sdc
/dev/nvme1n1
```

Always verify using:

```bash
lsblk
```

before running destructive commands.

---

## fstab Best Practice

For data disks:

Recommended:

```
UUID=<filesystem-uuid>
```

Avoid depending on:

```
/dev/sdX
```

or old LVM device identifiers:

```
/dev/disk/by-id/dm-uuid-LVM-*
```

---

## Downtime Estimate

Typical downtime:

| Task | Time |
|---|---:|
| Stop services | 2 min |
| Shutdown and disk replacement | 5 min |
| Create LVM and filesystem | 5 min |
| Mount and validation | 3 min |

Total estimated downtime:

```
~15 minutes
```
