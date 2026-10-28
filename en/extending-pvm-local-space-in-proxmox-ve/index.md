# Extending pvm-local Space in Proxmox VE


# Extending pvm-local Space in Proxmox VE

## Reason

The `proxmox ve` I installed earlier on the `DIY E3` server used a `120G` `SSD`. Back then, being young and naive, I also carved a `30g` `data` partition out of the `pve` logical volume. Now I want to use `local` for `glusterfs`, so I wanted to merge the space in `data` into `root`.

## Steps

### First, copy out the contents of the `data` partition

- First delete what you don't need on the `pve` `webui`; for what you do need, copy it out first. My `data` is mounted at `/mnt/data`:

```shell
sudo rsync -ar --info=progress2 /mnt/data/ /var/lib/vz
```

- After copying is done, remove the `data` logical volume:

```shell
sudo umount /mnt/data
sudo lvdisplay
```

- Take a look at the result:

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

- Remove it:

```shell
sudo lvremove pve/data

# Logical volume "data" successfully removed
```

- Check the `vg` space with `sudo vgdisplay`:

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

- As you can see, there is `30.39 Gib` of space. Now merge it into `pve-root`:
- `sudo lvresize -l +100%FREE /dev/pve/root`:

```shell
  Size of logical volume pve/root changed from 72.70 GiB (18612 extents) to <103.10 GiB (26393 extents).
  Logical volume pve/root successfully resized.`
```

- It becomes `103.10 GiB`.

- Extend the filesystem space: `sudo resize2fs -p /dev/mapper/pve-root`
- Result:

```shell
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/pve-root is mounted on /; on-line resizing required
old_desc_blocks = 10, new_desc_blocks = 13
The filesystem on /dev/mapper/pve-root is now 27026432 (4k) blocks long.
```

- Check the space with `df -h`:

```shell
Filesystem                 Size  Used Avail Use% Mounted on
udev                        11G     0   11G   0% /dev
tmpfs                      2.2G  6.5M  2.2G   1% /run
/dev/mapper/pve-root       102G   42G   56G  43% /
....
```

Done.

## (Addendum) Adding another disk to the `pve` `volume group`

I had another `120G` `SSD` that I also wanted to put in. This disk had already been added to the `vg`; `sudo pvdisplay`:

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

### Add it to the `vg`

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

As you can see, there is now `119.24G` of free space

### Extend the `lv`

- lvresize

```shell
sudo lvresize -l +100%FREE /dev/pve/root`
### output:
  Size of logical volume pve/root changed from <119.24 GiB (30525 extents) to <222.34 GiB (56918 extents).
  Logical volume pve/root successfully resized.
```

- Extend the filesystem space: `sudo resize2fs -p /dev/mapper/pve-root`
- Result:

```shell
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/pve-root is mounted on /; on-line resizing required
old_desc_blocks = 13, new_desc_blocks = 28
The filesystem on /dev/mapper/pve-root is now 58284032 (4k) blocks long.
```

Finished

## Summary

I feel that `lvm` is actually quite convenient for space management. It's just that its `pv`, `vg`, and `lv` concepts are confusing at first, so there's a bit of a learning curve to get started — not as straightforward as running `fdisk`, formatting, and mounting directly.

