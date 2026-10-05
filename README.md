# Android 11 Reports an exFAT SD Card as Corrupted After a ROM Downgrade

A cautious, no-formatting-first workflow for diagnosing a removable SD card that Android/Vold reports as **corrupted** or **unmountable** after a ROM change, interrupted reboot, or unclean shutdown.

> **Read this first:** Filesystem repair can cause data loss. If the data matters, stop and make a sector-level image or a full file backup before deleting anything or running a repair. Commands below use example device names and mount paths. Identify the correct partition on your own device every time. A wrong block device can destroy data on a different disk.

This guide is for rooted Android devices with an exFAT-formatted card and the necessary kernel/tools. Android ROMs differ; a command may be absent or behave differently on your device.

## What may have happened

A downgrade or interrupted shutdown can leave a removable filesystem marked dirty or with inconsistent metadata. Apps that write files—such as downloaders, browsers, media apps, or readers—may leave incomplete temporary data if the device reboots while a write is in progress. exFAT tracks directory entries and allocated clusters separately, so an interrupted update can leave references that do not agree.

Android's storage service (`vold`) checks and mounts public storage. If its check or mount fails, Settings may call the card “corrupted” or show it as `unmountable`. That message alone does not prove that the entire card or its contents are lost. Conversely, a successful manual mount does not prove the card is healthy: back up important data before attempting changes.

## Before you begin

- Keep the phone powered and avoid rebooting or removing the card during diagnosis.
- If possible, copy important files elsewhere first. If the card is unstable, disconnects, or reports I/O errors, stop repeated checks and prioritize imaging/recovery.
- Use a root shell. Depending on the setup, this may be through `adb shell` followed by `su`, or a trusted on-device root terminal.
- Substitute the verified partition everywhere. `/dev/block/mmcblk0p1` below is only an example.
- Do not run filesystem checks while the partition is mounted.

Example root-shell setup:

```sh
adb shell
su
```

## 1. Identify the disk and partition

Ask Android which disks and volumes it detects:

```sh
sm list-disks
sm list-volumes all
```

Example output might include a disk such as `disk:179,0` and a public volume such as `public:179,1 unmountable …`. These identifiers describe Android's view; they do not by themselves establish the Linux block-device path.

Inspect the kernel's partition list:

```sh
cat /proc/partitions
```

You may see a whole device and a partition, for example:

```text
mmcblk0
└── mmcblk0p1
```

Check the device links to correlate Android's disk/partition with block devices:

```sh
ls -l /dev/block
ls -l /dev/block/by-name 2>/dev/null
```

Some devices expose removable storage under a different path or name. Confirm the size and partition relationship; do not assume the example names match your phone. Set an example variable only after verifying it:

```sh
PART=/dev/block/mmcblk0p1
```

## 2. Confirm the filesystem and available exFAT support

Identify the partition:

```sh
blkid "$PART"
```

For this guide, the output should identify `TYPE="exfat"`. If it reports another filesystem, stop and use guidance for that filesystem instead.

Check whether the kernel advertises exFAT support and whether an exFAT checker exists:

```sh
cat /proc/filesystems | grep -i exfat
command -v fsck.exfat
ls -l /system/bin/*exfat* 2>/dev/null
```

ROMs may include the checker at `/system/bin/fsck.exfat` even if it is not on `PATH`. In the commands below, use the actual path on your device. The presence of `mkfs.exfat` does not mean it should be used: `mkfs` creates a new filesystem and can overwrite the existing filesystem metadata.

## 3. Run a non-modifying filesystem check

First make sure the partition is not mounted:

```sh
mount | grep -F "$PART"
```

If it is mounted, stop and cleanly unmount it before continuing. Then run the check-only mode:

```sh
/system/bin/fsck.exfat -n "$PART"
```

`-n` is intended to inspect without making repairs. Confirm the options supported by the checker installed on your ROM if needed (`fsck.exfat -h`). Save the complete output. It may report a clean volume or list inconsistencies such as file entries whose cluster allocation disagrees with the allocation bitmap.

A clean result is useful, but not a guarantee of healthy flash media. Errors, I/O failures, or repeated changes between runs are reasons to back up/image the card and investigate its health before proceeding.

## 4. Temporarily mount read-only and verify the data

Create a temporary mount point and mount the verified partition read-only:

```sh
mkdir -p /mnt/sdcard_check
mount -t exfat -o ro "$PART" /mnt/sdcard_check
```

If the mount succeeds, inspect the top-level folders and a few known files:

```sh
ls -lah /mnt/sdcard_check
```

If the data is readable, copy important files to another device/storage location before attempting deletion or repair. A read-only mount prevents ordinary writes through this mount and is the right first step for inspection.

If the mount fails, do not try random mount options or format the card. Record the exact error and preserve the card for recovery or a careful next diagnosis.

## 5. Remove only a positively identified expendable temporary item (optional)

Skip this step unless all of the following are true:

1. You have a backup of important data, or you accept the risk of losing the selected item.
2. The checker points to a specific affected entry, or you independently recognize it as incomplete temporary data.
3. You have confirmed the exact path is on this SD card and that it is safe to discard.

The previous mount is read-only. Do not assume deletion will work through it. If your device permits remounting this filesystem read-write, first unmount the read-only mount and remount explicitly as read-write:

```sh
umount /mnt/sdcard_check
mount -t exfat -o rw "$PART" /mnt/sdcard_check
```

Verify that this is the expected card and inspect the exact target and its parent before removing anything:

```sh
mount | grep -F /mnt/sdcard_check
ls -lah /mnt/sdcard_check
ls -lah "/mnt/sdcard_check/path/to/parent"
ls -ld "/mnt/sdcard_check/path/to/known-temporary-item"
```

Only after confirming the exact target, remove that one known expendable file or directory. Prefer a narrowly scoped `rm` or `rm -r` with a fully quoted exact path; avoid broad wildcards and avoid `rm -rf`:

```sh
rm -r "/mnt/sdcard_check/path/to/known-temporary-item"
```

Then verify that the intended item is gone and no neighboring data was selected:

```sh
ls -lah "/mnt/sdcard_check/path/to/parent"
```

If the filesystem will not mount read-write, do not force it. Skip deletion and proceed only with appropriate backup and repair advice. Deleting a visible file is not itself proof that filesystem metadata is repaired.

## 6. Unmount cleanly and check the filesystem again

Flush pending writes and unmount the temporary mount:

```sh
sync
umount /mnt/sdcard_check
```

Confirm it is no longer mounted before running the checker:

```sh
mount | grep -F "$PART"
```

No matching mount should remain. Re-run the non-modifying check and compare its output with the first run:

```sh
/system/bin/fsck.exfat -n "$PART"
```

A clean result means the checker currently sees no inconsistencies. If errors remain, do not repeat deletion against guessed files. Back up/image the card and assess whether a repair is appropriate.

## 7. Optional interactive repair

Use repair mode only after backing up important data, confirming the exact partition, and reading the help for the installed `fsck.exfat` version. Repair options vary by implementation. If that version supports `-r` for interactive repair, it may be used on an **unmounted** partition:

```sh
/system/bin/fsck.exfat -r "$PART"
```

Read every prompt. Do not automatically answer `y` to all questions. A prompt to truncate or discard a file can mean that file's contents will be lost. Stop if you cannot tell what a proposed repair will do. Do not substitute an automatic “yes” mode for understanding the repair.

After any repair, verify the partition is still unmounted and run check-only mode again:

```sh
/system/bin/fsck.exfat -n "$PART"
```

## 8. Let Android/Vold mount the card

Once the temporary mount is gone and the check is clean, ask Android to report the current state:

```sh
sm list-disks
sm list-volumes all
```

If the public volume still shows `unmountable`, you can ask Android's storage manager to mount that volume using the identifier reported by `sm list-volumes all` (example only):

```sh
sm mount public:179,1
sm list-volumes all
```

Do not leave the partition manually mounted while asking Vold to mount it. If Android changes the state to `mounted`, verify the card through Android's normal file manager.

## If Vold still reports `unmountable`

If `fsck.exfat -n` reports clean but Android still refuses to mount the card, avoid further filesystem changes until you know why. The remaining issue may involve Vold's mount path, ROM-specific exFAT integration, permissions/SELinux, or a kernel/storage compatibility problem.

Capture the result of the mount attempt and relevant system logs. For example:

```sh
logcat -c
sm mount public:179,1
logcat -b all -d | grep -iE 'vold|VolumePublic|fsck|exfat|mount|denied|avc' | tail -100
```

`public:179,1` is only an example: use the identifier your device reports. On some Android builds, `logcat` options or access differ. Kernel messages may also help:

```sh
dmesg | grep -iE 'exfat|mmc|mount|denied|avc'
```

Record the exact ROM/build, checker output, mount error, and relevant log lines. If logs indicate I/O errors or the card repeatedly disconnects, stop repair attempts and prioritize a backup/image or card replacement. Otherwise, investigate the ROM's storage configuration or seek ROM-specific support. A filesystem that checks clean does not rule out a Vold or driver issue.

## Common commands at a glance

```sh
sm list-disks
sm list-volumes all
cat /proc/partitions
blkid "$PART"
cat /proc/filesystems | grep -i exfat
/system/bin/fsck.exfat -n "$PART"
mount -t exfat -o ro "$PART" /mnt/sdcard_check
sync
umount /mnt/sdcard_check
```

Use the actual checker path, partition, and volume identifier found on your device.

## Do not do these things

- **Do not format** just because Android displays “corrupted.” Formatting destroys the existing filesystem structure and can make recovery harder.
- **Do not run `mkfs.exfat`** on a card containing data. It creates a new filesystem.
- **Do not run `rm -rf` with a broad, guessed, or unverified path.** In a root shell, a path mistake can delete far more than intended. Never target `/`, `/storage`, or an assumed mount point.
- **Do not delete files simply because their names look temporary.** Delete only an exact item you have verified is expendable.
- **Do not run `fsck.exfat` on a mounted partition.** Unmount it first.
- **Do not use automatic repair answers blindly.** Repair can discard file data.
- **Do not assume `/sdcard` or `/storage/emulated/0` is the physical removable card.** Those commonly refer to emulated internal storage. Verify the actual mount and device.
- **Do not keep retrying on a failing card.** Repeated reads/writes can reduce recovery chances if the media is physically failing.

## Workflow summary

```text
Identify disk and partition
        ↓
Confirm exFAT and tool support
        ↓
Run fsck.exfat -n while unmounted
        ↓
Mount read-only and verify/copy data
        ↓
Optionally remove one known expendable temporary item
        ↓
Unmount cleanly and run fsck.exfat -n again
        ↓
If needed, consider supported interactive repair after backup
        ↓
Ensure manual mount is gone; let Vold mount the volume
        ↓
If still unmountable, inspect Vold/ROM logs instead of formatting
```

This is a recovery workflow, not a guarantee. The safest priority is to preserve a copy of important data before changing the filesystem.
