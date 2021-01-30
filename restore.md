# Restoring a Pi 4 from Backup

I have:
* A Raspberry Pi 4
* A broken µSD card
* Backups of the `/` and `/boot` filesystems
* A new µSD card

I want:
* A bootable µSD card with a restore of my backup

# Investigation

## `/boot`

`/boot` seems to be the first partition and contains a kernel image and a bunch of other stuff which I guess is to do with the boot chain.
Also an `issue.txt` mentioning [pi-gen](https://github.com/RPi-Distro/pi-gen).

## Partition UUIDs

These come from [03-set-partuuid/00-run.sh](https://github.com/RPi-Distro/pi-gen/blob/master/export-image/03-set-partuuid/00-run.sh) which picks them out of the raw image.
It takes 4 bytes at offset 440 (0x01B8), which [Master boot record](https://en.wikipedia.org/wiki/Master_boot_record) documents as a 32-bit disk signature.
The resulting 8-digit hex value need to be patched into `/etc/fstab` and `/boot/cmdline.txt`.

## Image Creation

This is done in [export-image/prerun.sh](https://github.com/RPi-Distro/pi-gen/blob/master/export-image/prerun.sh):
* The disk uses MBR partitioning
* `/boot` and `/` are aligned on 4MB boundaries, and are both primary partitions.
* `/boot` is 256MB FAT32, name `boot`
* `/` is ext4 and has features `^huge_file,metadata_csum,64bit`.

As far as I can see neither are marked bootable.

## Restoration

* Use your favorite tool to to create partitions. I had to manually patch in a disk signature.
* `DEV=/dev/sdX`
* `mkdosfs -n boot -F 32 -v ${DEV}1`
* `mkfs.ext4 -L rootfs -O '^huge_file,metadata_csum,64bit' ${DEV}2`
* `mount ${DEV}2 /mnt`
* `cp -a /path/to/root/. /mnt/.`
* `mount ${DEV}1 /mnt/boot`
* `cp -a /path/to/boot/. /mnt/boot/.`
* `dd status=none if=/dev/sde  skip=440 bs=1 count=4 | xxd -e |cut -f 2 -d' '` to find the disk signature
* `emacs /mnt/boot/cmdline.txt /mnt/etc/fstab` and adjust the UUIDs
* `umount /mnt/boot`
* `umount /mnt`

