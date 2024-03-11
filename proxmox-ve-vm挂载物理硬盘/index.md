# proxmox ve VM挂载物理硬盘


# proxmox ve VM挂载物理硬盘

先找到要挂载的硬盘的`id`:

```shell
lsblk |awk 'NR==1{print $0" DEVICE-ID(S)"}NR>1{dev=$1;printf $0" ";system("find /dev/disk/by-id -lname \"*"dev"\" -printf \" %p\"");print "";}'|grep -v -E 'part|lvm'
```

结果类似:

```shell
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT DEVICE-ID(S)
loop0          7:0    0    20G  0 loop
loop1          7:1    0    80G  0 loop
loop2          7:2    0    50G  0 loop
sda            8:0    0   3.6T  0 disk   /dev/disk/by-id/ata-WDC_WD40EJRX-XXXXXXX_WD-WWWWW0XXXXXX /dev/disk/by-id/wwn-0x5000000000000
```

比如要挂载`/dev/disk/by-id/wwn-0x5000000000000`,`VM`编号为: `103`,执行如下操作:

```shell
# -scsi2为第二块scsi设备,如果已经被占用,就换一个编号
qm set 103 -scsi2 /dev/disk/by-id/wwn-0x5000000000000

#最后会输出:
# update VM 103: -scsi2 /dev/disk/by-id/wwn-0x5000000000000
```

设置完成后,查询一下:

```shell
grep wwn-0x5000000000000 /etc/pve/qemu-server/103.conf
```

拨出磁盘:

```shell
qm unlink 103 --idlist scsi2
```

