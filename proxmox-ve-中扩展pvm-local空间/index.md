# proxmox ve 中扩展pvm-local空间


# proxmox ve 中扩展pvm-local空间

## 原因

之前在`DIY E3`服务器上安装的`proxmox ve`,用的一张`120G`的`SSD`,当时年纪轻,把`pve`的逻辑分区还分了一个`30g`的`data`分区出来了,现在想用`local`做`glusterfs`,那么就想把`data`中的空间合并到`root`中去了.

## 步骤

### 先拷出`data`分区里面的东西

- 先在`pve`的`webui`上删除不需要的,对于需要的,先拷贝出来,我的`data`是挂载在`/mnt/data`中:

```shell
sudo rsync -ar --info=progress2 /mnt/data/ /var/lib/vz
```

- 拷贝完成后,删除`data` 逻辑分区:

```shell
sudo umount /mnt/data
sudo lvdisplay
```

- 看一下结果:

```shell
 --- Logical volume ---
  LV Path                /dev/pve/data
  LV Name                data
  VG Name                pve
  LV UUID                xxxx
  LV Write Access        read/write
  LV Creation host, time pve, 2020-12-13 12:19:01 +0800
  LV Status              available
  # open                 0
  LV Size                30.39 GiB
  Current LE             7781
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:2
```

- 删除:

```shell
sudo lvremove pve/data

# Logical volume "data" successfully removed
```

- 查看`vg`空间, `sudo vgdisplay`:

```shell
  --- Volume group ---
  VG Name               pve
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  73
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <106.60 GiB
  PE Size               4.00 MiB
  Total PE              27289
  Alloc PE / Size       19508 / 76.20 GiB
  Free  PE / Size       7781 / 30.39 GiB
  VG UUID               xxxx
```

- 可以看到有`30.39 Gib`的空间,现在把他合并到`pve-root`:
- `sudo lvresize -l +100%FREE /dev/pve/root`:

```shell
  Size of logical volume pve/root changed from 72.70 GiB (18612 extents) to <103.10 GiB (26393 extents).
  Logical volume pve/root successfully resized.`
```

- 变成`103.10 GiB`了.

- 扩展文件系统空间: `sudo resize2fs -p /dev/mapper/pve-root`
- 结果:

```shell
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/pve-root is mounted on /; on-line resizing required
old_desc_blocks = 10, new_desc_blocks = 13
The filesystem on /dev/mapper/pve-root is now 27026432 (4k) blocks long.
```

- 看看空间: `df -h`:

```shell
Filesystem                 Size  Used Avail Use% Mounted on
udev                        11G     0   11G   0% /dev
tmpfs                      2.2G  6.5M  2.2G   1% /run
/dev/mapper/pve-root       102G   42G   56G  43% /
....
```

成功了

## (补充)将另一块硬盘加入到`pve`的`volume group`中去

之前还有一块`120G`的`SSD`,想把他也装进去,这块硬盘已经加到`vg`里面了,`sudo pvdisplay`:

```shell
  --- Physical volume ---
  PV Name               /dev/sdh3
  VG Name               pve
  PV Size               106.60 GiB / not usable <2.55 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              27289
  Free PE               0
  Allocated PE          27289
  PV UUID               xxxx

  "/dev/sdf1" is a new physical volume of "119.24 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdf1
  VG Name
  PV Size               119.24 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               xxxx
```

### 加入`vg`

```shell
sudo vgextend pve /dev/sdf1`
#  Volume group "pve" successfully extended`

sudo vgdisplay
#  output:
  --- Volume group ---
  VG Name               pve
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  75
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               <225.84 GiB
  PE Size               4.00 MiB
  Total PE              57814
  Alloc PE / Size       27289 / <106.60 GiB
  Free  PE / Size       30525 / <119.24 GiB
  VG UUID               qTOzVc-TkpO-iokP-8JGO-3MNo-4Id0-TAsIYi
```

可以看到已经有`119.24G`空闲空间了

### 扩展`lv`

- lvresize

```shell
sudo lvresize -l +100%FREE /dev/pve/root`
### output:
  Size of logical volume pve/root changed from <119.24 GiB (30525 extents) to <222.34 GiB (56918 extents).
  Logical volume pve/root successfully resized.
```

- 扩展文件系统空间: `sudo resize2fs -p /dev/mapper/pve-root`
- 结果:

```shell
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/pve-root is mounted on /; on-line resizing required
old_desc_blocks = 13, new_desc_blocks = 28
The filesystem on /dev/mapper/pve-root is now 58284032 (4k) blocks long.
```

完成

## 总结

感觉 `lvm`在空间管理这块还是蛮方便的,只是,他的`pv`, `vg`, `lv`,一开始太绕,入门有点门槛,不像直接`fdisk`后格式化挂载那么方便.

