# Mounting a Physical Disk to a Proxmox VE VM


# Mounting a Physical Disk to a Proxmox VE VM

First, find the `id` of the disk you want to mount:

```shell
lsblk |awk 'NR==1{print $0" DEVICE-ID(S)"}NR>1{dev=$1;printf $0" ";system("find /dev/disk/by-id -lname \"*"dev"\" -printf \" %p\"");print "";}'|grep -v -E 'part|lvm'
```

The output looks similar to this:

```shell
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT DEVICE-ID(S)
loop0          7:0    0    20G  0 loop
loop1          7:1    0    80G  0 loop
loop2          7:2    0    50G  0 loop
sda            8:0    0   3.6T  0 disk   /dev/disk/by-id/ata-WDC_WD40EJRX-XXXXXXX_WD-WWWWW0XXXXXX /dev/disk/by-id/wwn-0x5000000000000
```

For example, to mount `/dev/disk/by-id/wwn-0x5000000000000` for a `VM` whose ID is `103`, run the following:

```shell
# -scsi2 is the second scsi device; if it is already taken, use another number
qm set 103 -scsi2 /dev/disk/by-id/wwn-0x5000000000000

# It will finally output:
# update VM 103: -scsi2 /dev/disk/by-id/wwn-0x5000000000000
```

After the setup is complete, verify it:

```shell
grep wwn-0x5000000000000 /etc/pve/qemu-server/103.conf
```

Detach the disk:

```shell
qm unlink 103 --idlist scsi2
```

