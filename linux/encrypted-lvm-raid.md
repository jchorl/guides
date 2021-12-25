# Encrypted LVM RAID
## What??
[LVM](https://wiki.archlinux.org/index.php/LVM) is compatible with LUKS to use encrypted drives with LVM. LVM also supports [RAID](https://wiki.archlinux.org/index.php/RAID). I wanted my content replicated (RAID1) across two encrypted drives.

![](https://2.bp.blogspot.com/-YQfGEJjE0D0/Wn_t96fSqYI/AAAAAAAAMgQ/olcY9HDTbc4vhkdfoTbB6HrGr0CVls6vgCLcBGAs/s1600/why-not-both-animated-gif-7.gif)

The Arch LVM docs are really amazing: https://wiki.archlinux.org/index.php/LVM

## How?
### Step 0: Figure out what drives you're dealing with
```bash
j@troy:~$ lsblk
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
...
sdb                   8:16   0 931.5G  0 disk
├─sdb1                8:17   0   200M  0 part
└─sdb2                8:18   0 931.3G  0 part
sdc                   8:32   0 931.5G  0 disk
├─sdc1                8:33   0   200M  0 part
└─sdc2                8:34   0 931.3G  0 part
```

### Step 1: Use fdisk to wipe the drives
```bash
j@troy:~$ sudo fdisk /dev/sdb
# g to create a new guid partition table
# n for new partition (defaults should be fine)
# w to write changes
```

### Step 2: Configure drive encryption
LUKS format the devices
```bash
j@troy:~$ sudo cryptsetup luksFormat /dev/sdb1 # NOTE to use sdb1, not sdb
WARNING!
========
This will overwrite data on /dev/sdb1 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/sdb1:
Verify passphrase:
```

Repeat for the other drive.

Lastly, open the drives so they can be used for LVM:
```bash
j@troy:~$ sudo cryptsetup open /dev/sdc1 cryptrootb
Enter passphrase for /dev/sdc1:
j@troy:~$ lsblk
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
...
sdb                   8:16   0 931.5G  0 disk
└─sdb1                8:17   0 931.5G  0 part
  └─cryptroota      253:3    0 931.5G  0 crypt
sdc                   8:32   0 931.5G  0 disk
└─sdc1                8:33   0 931.5G  0 part
  └─cryptrootb      253:4    0 931.5G  0 crypt
```

### Step 3: Set up the LVM
First up, create the PVs:
```bash
j@troy:~$ sudo pvcreate /dev/mapper/cryptroota /dev/mapper/cryptrootb
  Physical volume "/dev/mapper/cryptroota" successfully created.
  Physical volume "/dev/mapper/cryptrootb" successfully created.
```

Create the VG:

_NOTE: I later renamed the volume using `sudo vgrename /dev/VolGroup00 /dev/vgsnails`, it's a good idea to pick a simple, descriptive name_

```bash
j@troy:~$ sudo vgcreate VolGroup00 /dev/mapper/cryptroota /dev/mapper/cryptrootb
  Volume group "VolGroup00" successfully created
```

Create the LVs:
```bash
j@troy:~$ sudo lvcreate --type raid1 --mirrors 1 -L 400G -n data VolGroup00 /dev/mapper/cryptroota /dev/mapper/cryptrootb
  Logical volume "data" created.
...
j@troy:~$ sudo lvcreate --type raid1 --mirrors 1 -l 100%FREE -n scrap VolGroup00 /dev/mapper/cryptroota /dev/mapper/cryptrootb
  Logical volume "scrap" created.
```

Now the drives should look as follows:
```bash
j@troy:~$ lsblk
NAME                             MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sdb                                8:16   0 931.5G  0 disk
└─sdb1                             8:17   0 931.5G  0 part
  └─cryptroota                   253:3    0 931.5G  0 crypt
    ├─VolGroup00-data_rmeta_0    253:5    0     4M  0 lvm
    │ └─VolGroup00-data          253:9    0   400G  0 lvm
    ├─VolGroup00-data_rimage_0   253:6    0   400G  0 lvm
    │ └─VolGroup00-data          253:9    0   400G  0 lvm
    ├─VolGroup00-backup_rmeta_0  253:10   0     4M  0 lvm
    │ └─VolGroup00-backup        253:14   0   400G  0 lvm
    ├─VolGroup00-backup_rimage_0 253:11   0   400G  0 lvm
    │ └─VolGroup00-backup        253:14   0   400G  0 lvm
    ├─VolGroup00-scrap_rmeta_0   253:15   0     4M  0 lvm
    │ └─VolGroup00-scrap         253:19   0 131.5G  0 lvm
    └─VolGroup00-scrap_rimage_0  253:16   0 131.5G  0 lvm
      └─VolGroup00-scrap         253:19   0 131.5G  0 lvm
sdc                                8:32   0 931.5G  0 disk
└─sdc1                             8:33   0 931.5G  0 part
  └─cryptrootb                   253:4    0 931.5G  0 crypt
    ├─VolGroup00-data_rmeta_1    253:7    0     4M  0 lvm
    │ └─VolGroup00-data          253:9    0   400G  0 lvm
    ├─VolGroup00-data_rimage_1   253:8    0   400G  0 lvm
    │ └─VolGroup00-data          253:9    0   400G  0 lvm
    ├─VolGroup00-backup_rmeta_1  253:12   0     4M  0 lvm
    │ └─VolGroup00-backup        253:14   0   400G  0 lvm
    ├─VolGroup00-backup_rimage_1 253:13   0   400G  0 lvm
    │ └─VolGroup00-backup        253:14   0   400G  0 lvm
    ├─VolGroup00-scrap_rmeta_1   253:17   0     4M  0 lvm
    │ └─VolGroup00-scrap         253:19   0 131.5G  0 lvm
    └─VolGroup00-scrap_rimage_1  253:18   0 131.5G  0 lvm
      └─VolGroup00-scrap         253:19   0 131.5G  0 lvm
```

### Step 4: Create the filesystems
```bash
j@troy:~$ sudo mkfs.ext4 /dev/vgsnails/backup
```

Repeat for all partitions

### Step 5: Mount the partition
```bash
j@troy:~$ ls /dev/vgsnails/
backup  data  scrap
j@troy:~$ sudo mount /dev/vgsnails/data /mnt
j@troy:~$ sudo touch /mnt/done
```

### Step 6: Mount the partitions at boot

[This](https://blog.tinned-software.net/automount-a-luks-encrypted-volume-on-system-start/) guide was very helpful!

First, generate some keys. They can be long and should be random. You need not remember these or save them anywhere else (except on the encrypted root drive of the server itself).
```bash
j@troy:~$ sudo lshw -short -C disk
H/W path         Device     Class          Description
======================================================
/0/a/0.0.0       /dev/sda   disk           250GB Samsung SSD 860
/0/b/0.0.0       /dev/sdb   disk           1TB ST1000VN002-2EY1
/0/c/0.0.0       /dev/sdc   disk           1TB WDC WD10EFRX-68F

j@troy:~$ sudo vim /etc/luks-keys/seagate-1tb-0
# paste randomly generated key

# OPTIONAL: remove newline that vim adds, for good measure, if there is one
j@troy:~$ sudo truncate -s -1 /etc/luks-keys/seagate-1tb-0

j@troy:~$ sudo vim /etc/luks-keys/wd-1tb-0
# paste (different) randomly generated key

# OPTIONAL: remove newline that vim adds, for good measure, if there is one
j@troy:~$ sudo truncate -s -1 /etc/luks-keys/wd-1tb-0
```

Add the keys to the drives:
```bash
j@troy:~$ sudo cryptsetup -v luksAddKey /dev/sdb1 /etc/luks-keys/seagate-1tb-0
j@troy:~$ sudo cryptsetup -v luksAddKey /dev/sdc1 /etc/luks-keys/wd-1tb-0
```

Grab the drive UUIDs:
```bash
j@troy:~$ sudo cryptsetup luksDump /dev/sdb1 | grep "UUID"
UUID:           3fa5b8f4-6220-4194-80ac-dba0923f74f9
j@troy:~$ sudo cryptsetup luksDump /dev/sdc1 | grep "UUID"
UUID:           9cfbca69-7176-4d0a-8b9c-68a0297d8d34
```

Add the UUIDs to `/etc/crypttab` so they will get decrypted on boot:
```bash
j@troy:~$ sudo cat /etc/crypttab
[sudo] password for j:
dm_crypt-0 UUID=8e867319-c056-4a18-be60-2db0b98c800e none luks
sdb1_crypt UUID=3fa5b8f4-6220-4194-80ac-dba0923f74f9 /etc/luks-keys/seagate-1tb-0 luks
sdc1_crypt UUID=9cfbca69-7176-4d0a-8b9c-68a0297d8d34 /etc/luks-keys/wd-1tb-0 luks
```

Finally, add the decrypted LVs to `/etc/fsstab` to ensure the volumes get mounted at boot:
```bash
j@troy:~$ sudo cat /etc/fstab
...
/dev/vgsnails/backup    /mnt/backup     ext4    defaults        0       0
/dev/vgsnails/data      /mnt/data       ext4    defaults        0       0
/dev/vgsnails/scrap     /mnt/scrap      ext4    defaults        0       0
```
