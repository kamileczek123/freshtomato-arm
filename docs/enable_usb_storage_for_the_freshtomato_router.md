# Enable USB Storage for the FreshTomato Router

If USB 2.0 and USB 3.0 ports used at the same time, OS attaches USB 3.0 device as `sda` and USB 2.0 device as `sdb`. Then the `sdb` device is used as **primary storage**, while `sda` can be used as **backup storage** to simplify disaster recovery in case the primary USB device will die. 

It is recommended to use USB 3.0 drive attached to USB 3.0 port on the router to assure higher read/writes than are available on USB 2.0 port and drive. When a slow drive or port is used, this is clearly felt when running the Entware applications.

This documentation covers use of `ext2` filesystem on USB storage from two reasons;
1. to extend available router memory by SWAP partition - it is required to run `python3` which is required by Ansible,
2. to keep consistency with FreshTomato OS BusyBox running.

## Setup SWAP and OPT partitions

Requires `parted` to be installed on the system, whithout Entware it is not possible on router, so the partition setup has to be done on another host with `parted` available.

1. Identify USB drive

```bash
fdisk -l
```

The result should be something like below example:

```bash
Disk /dev/sda: 3.73 GiB, 4009754624 bytes, 7831552 sectors
Disk model: USB 3.0 FD
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

The USB drive has been identified as `/dev/sda`, if different identifier adjust the commands below.

2. Create partitions

```bash
parted -a optimal /dev/sda

mklabel GPT
unit GB
mkpart primary linux-swap 0.00GB 1.00GB
mkpart primary ext2 1.00GB 100%
p
q
```

3. Format partitions

```bash
mkswap -L _SWAP /dev/sda1
mkfs.ext2 -L _OPT /dev/sda2
```

4. Check the results

```bash
blkid

# output
/dev/sdc2: LABEL="_OPT" UUID="6d0c01ef-346e-4115-8858-78c815936f06" BLOCK_SIZE="4096" TYPE="ext2" PARTLABEL="primary" PARTUUID="e9ca93c6-d742-4983-b979-fc22120dd9c5"
/dev/sdc1: LABEL="_SWAP" UUID="898a89cd-a087-4e1a-8db9-c01d17f7dbf0" TYPE="swap" PARTLABEL="primary" PARTUUID="076065ae-c9bd-4982-b773-910d7e5069c9"
```

## Enable USB Storage

This is temporary setup for initial Entware installation. Final USB Storage configuration, which overwrite this has to be done using Ansible and `usb_support` tag after the Enthware is installed.

The temporary setup can be done manually via Web UI or alternatively via commandline, see below.
The assumption is that USB pendrives are already formatted and attached to the router.

## Web UI Setup

1. Configure USB parameters via Web UI, it is available in *USB and NAS > USB Support* page.

Core USB Support: `v`
USB 3.0 Support: `v`
USB 2.0 Support: `v`
USB 1.1 Support: `o`:OHCI `o`:UHCI

USB Printer Support: `o`
Bidirectional copying: `o`

USB Storage Support: `v`
File Systems Support: `v`:Ext2 / Ext3 / Ext4 `o`:NTFS `v`:FAT `v`:exFAT `o`:HFS/HFS+ `o`:ZFS
NTFS Driver: `Open NTFS-3G driver`
HFS/HFS+ Driver: `Open HFS/HFS+ driver`
Automount: `v` (Automatically mount all partitions to sub-directories in /mnt.)
Run after mounting: 
Run before unmounting:
HDD Spindown: `o`
3G/4G Modem Support: `o`
Run APCUPSD Deamon: `o`
Custom Config File: `o`   (located /etc/apcupsd.conf)
Hotplug script:

2. Save changes and Reboot the router with USB drive attached to allow on initial partitions mount.

## Commandline Setup

1. Login to the router via terminal
2. Call the followig `nvram` commands

```bash
nvram set usb_enable='1'
nvram set usb_usb3='1'
nvram set usb_usb2='1'
nvram set usb_ohci='0'
nvram set usb_uhci='-1'
nvram set usb_storage='1'
nvram set usb_fs_ext4='1'
nvram set usb_fs_fat='1'
nvram set usb_fs_exfat='1'
nvram set usb_automount='1'
nvram commit
service usb restart
```

3. Reboot the router with USB drive attached to allow on initial partitions mount.

## Mount OPT and SWAP partitions

1. Login to the router via terminal
2. Check if partitions are available

```bash
blkid

# output
/dev/sda2: LABEL="_OPT" UUID="6d0c01ef-346e-4115-8858-78c815936f06"
/dev/sda1: LABEL="_SWAP" UUID="898a89cd-a087-4e1a-8db9-c01d17f7dbf0"
```

3. Mount partitions

```bash
mount -t ext2 -o rw,noatime LABEL=_OPT /opt
/sbin/swapon LABEL=_SWAP
```

4. Check if partitions are mounted correctly

- OPT partition

```bash
df -h | grep /dev/sda

# output
/dev/sda2   2.8G   4.2M   2.6G   0%   /tmp/mnt/_OPT
/dev/sda2   2.8G   4.2M   2.6G   0%   /opt
```

- SWAP partition

```bash
cat /proc/swaps

# ouput
Filename    Type        Size     Used  Priority
/dev/sda1   partition   975868   0     -1
```
