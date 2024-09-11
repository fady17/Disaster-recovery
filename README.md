# Disaster Recovery: Creating a Full System Backup ISO Image

This tutorial will guide you through the process of creating a bootable ISO image that contains a complete backup of your entire system, including all folders and installed software.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Preparation](#preparation)
3. [Creating the Backup](#creating-the-backup)
4. [Creating the ISO Image](#creating-the-iso-image)
5. [Finalizing the ISO](#finalizing-the-iso)
6. [Important Notes](#important-notes)
7. [Restoring from the Backup](#restoring-from-the-backup)

## Prerequisites

Before you begin, ensure you have:

- A running Ubuntu or Debian-based system
- Root or sudo access
- External storage with enough free space to hold your entire system (recommend at least 1.5x your current used space)
- LiveCD or USB with Ubuntu (for creating the backup while the system is not running)

## Preparation

1. Boot from a LiveCD/USB
   - This ensures all system files are not in use and can be copied cleanly.

2. Open a terminal

3. Update the package list and install necessary tools:
   ```bash
   sudo apt update
   sudo apt install squashfs-tools genisoimage
   ```

## Creating the Backup

1. Mount your system drive and the external storage:
   ```bash
   sudo mkdir /mnt/system /mnt/backup
   sudo mount /dev/sdXY /mnt/system  # Replace sdXY with your system partition
   sudo mount /dev/sdZW /mnt/backup  # Replace sdZW with your backup drive
   ```

2. Create a full system backup:
   ```bash
   sudo rsync -avxHAX --progress /mnt/system/ /mnt/backup/system-backup/
   ```
   This command copies all files, preserving permissions, timestamps, and other attributes.

## Creating the ISO Image

1. Create a SquashFS of your backup:
   ```bash
   sudo mksquashfs /mnt/backup/system-backup /mnt/backup/filesystem.squashfs -comp xz -Xbcj x86
   ```

2. Set up the ISO directory structure:
   ```bash
   mkdir -p /mnt/backup/iso/{casper,isolinux,install}
   ```

3. Copy the kernel and initrd from your system:
   ```bash
   cp /mnt/system/boot/vmlinuz-* /mnt/backup/iso/casper/vmlinuz
   cp /mnt/system/boot/initrd.img-* /mnt/backup/iso/casper/initrd
   ```

4. Copy isolinux files:
   ```bash
   cp /usr/lib/ISOLINUX/isolinux.bin /mnt/backup/iso/isolinux/
   cp /usr/lib/syslinux/modules/bios/* /mnt/backup/iso/isolinux/
   ```

5. Create a menu configuration file:
   ```bash
   cat << EOF > /mnt/backup/iso/isolinux/isolinux.cfg
   DEFAULT live
   LABEL live
     menu label ^Start or install Ubuntu
     kernel /casper/vmlinuz
     append  file=/cdrom/preseed/ubuntu.seed boot=casper initrd=/casper/initrd quiet splash ---
   LABEL check
     menu label ^Check disc for defects
     kernel /casper/vmlinuz
     append  boot=casper integrity-check initrd=/casper/initrd quiet splash ---
   LABEL memtest
     menu label Test ^memory
     kernel /install/mt86plus
   LABEL hd
     menu label ^Boot from first hard disk
     localboot 0x80
   EOF
   ```

6. Create the ISO image:
   ```bash
   sudo genisoimage -r -V "System Backup" -cache-inodes -J -l \
     -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot \
     -boot-load-size 4 -boot-info-table \
     -o /mnt/backup/full-system-backup.iso /mnt/backup/iso
   ```

## Finalizing the ISO

Make the ISO hybrid (bootable from USB devices):
```bash
isohybrid /mnt/backup/full-system-backup.iso
```

## Important Notes

- This process creates a full, exact copy of your system. It will include all personal files, so be cautious about sharing or storing this ISO.
- The resulting ISO file will be large, potentially several gigabytes or more, depending on your system.
- This backup method is best for personal use or disaster recovery, not for distribution.
- Ensure you have sufficient rights to backup and restore all software on your system.
- Test the ISO in a virtual machine before relying on it for critical backups.
- Regular backups are still recommended, as this ISO represents a point-in-time backup.

## Restoring from the Backup

To restore your system from this ISO:

1. Boot from the ISO
2. Use a live environment tool like `gparted` to set up your target drive's partitions
3. Mount your target drive and the ISO's filesystem
4. Use `rsync` to copy the files from the ISO to your target drive
5. Reinstall GRUB to make the system bootable
6. Reboot into your restored system

Remember to adjust hardware-specific configurations (like `/etc/fstab`) if restoring to different hardware.
