# Notes on Deploying a Two-Node HA Cluster and GlusterFS Distributed File System on Proxmox VE


# Deploying a Two-Node HA Cluster and GlusterFS Distributed File System on Proxmox VE

**Make sure both pve nodes run the same version**

Two `proxmox ve` nodes, enabling high availability with automatic `vm` and `lxc` migration

## Edit the `hosts` file

In `/etc/hosts` on both `pve` machines, add the following `host` entries:

```text
192.168.1.229 pve1.local pve1
192.168.1.230 pve2.local pve2
192.168.1.229 gluster1
192.168.1.230 gluster2
```

## Install `glusterfs`

The following operations are performed on both machines, referred to here as `pve1` and `pve2`

```shell
wget -O - https://download.gluster.org/pub/gluster/glusterfs/9/rsa.pub | apt-key add -

DEBID=$(grep 'VERSION_ID=' /etc/os-release | cut -d '=' -f 2 | tr -d '"')
DEBVER=$(grep 'VERSION=' /etc/os-release | grep -Eo '[a-z]+')
DEBARCH=$(dpkg --print-architecture)
echo deb https://download.gluster.org/pub/gluster/glusterfs/LATEST/Debian/${DEBID}/${DEBARCH}/apt ${DEBVER} main > /etc/apt/sources.list.d/gluster.list

apt update && apt install -y glusterfs-server
```

**Make sure the gluster versions are the same**

```shell
gluster --version
```

If they are the same, upgrade them. One of my machines was `9.2`, and even after adding the source it wouldn't upgrade directly to `10.3`, so you need to do this:

```shell
apt-get list -a glusterfs-server
```

It will list all available versions; then install the corresponding one

```shell
apt-get install glusterfs-server="10.3-1"
```

On `pve1`, edit `nano /etc/glusterfs/glusterd.vol`, and under `option transport.socket.listen-port 24007` add:

```text
    option transport.rdma.bind-address gluster1
    option transport.socket.bind-address gluster1
    option transport.tcp.bind-address gluster1
```

On `pve2`, edit `nano /etc/glusterfs/glusterd.vol`, and under `option transport.socket.listen-port 24007` add:

```text
    option transport.rdma.bind-address gluster2
    option transport.socket.bind-address gluster2
    option transport.tcp.bind-address gluster2
```

### Start the service

```shell
systemctl enable glusterd.service
systemctl start glusterd.service
```

**Important: on `pve2` you need to run the command to join the cluster**

```shell
gluster peer probe gluster1
```

It's OK once it shows: `peer probe: success`

### Add a `volume`

Here it is created directly on `zfs`; make sure both `pve` nodes use the same path to create the `volume`, which is `/tank/gfs` here
**Note: if it's zfs, don't use the mount root itself, but use a directory.** In the example below, `/tank/gfs` is a `zfs` mount, and a subdirectory `s` is created under it to serve as the `volume`

```shell
gluster volume create VMS replica 2 gluster1:/tank/gfs/s gluster2:/tank/gfs/s

gluster vol start VMS
```

### Check the status

```shell
gluster vol info VMS
gluster vol status VMS
```

### Add the mount

Do this on both `pve` nodes
```shell
mkdir /vms
```

Edit `pve1`'s `/etc/fstab` and add

```text
gluster1:VMS /vms glusterfs defaults,_netdev,x-systemd.automount,backupvolfile-server=gluster2 0 0
```

Edit `pve2`'s `/etc/fstab` and add

```text
gluster2:VMS /vms glusterfs defaults,_netdev,x-systemd.automount,backupvolfile-server=gluster1 0 0
```

Reboot both `pve` nodes to let `/mnt` mount

#### Mount without rebooting both `pve` nodes

```shell
mount /vms
```

## Solving the `split-brain` problem

With two nodes, `gluster` can run into a `split-brain` problem — the two nodes have equal votes and neither will listen to the other. The solution is as follows:

```shell
gluster vol set vms cluster.heal-timeout 5
gluster volume heal vms enable
gluster vol set vms cluster.quorum-reads false
gluster vol set vms cluster.quorum-count 1
gluster vol set vms network.ping-timeout 2
gluster volume set vms cluster.favorite-child-policy mtime
gluster volume heal vms granular-entry-heal enable
gluster volume set vms cluster.data-self-heal-algorithm full
```

## Setting up the two-node `pve` cluster

Create on the first, join on the second — nothing much to say

### Create a shared directory

In `Storage` under `DataCenter`, click `Add`, fill in `/vms` for `Directory`, and **check share**

### HA setup

Edit `/etc/pve/corosync.conf`

Add to `quorum` so it becomes like this:

```json
quorum {
    provider: corosync_votequorum
    expected_votes: 1
    two_node: 1
}
```
Here, `expected_votes` indicates the desired number of nodes, and `two_node: 1` means there are only two nodes. There is also a `wait_for_all: 0`, [NOTES: enabling two_node: 1 automatically enables wait_for_all. It is still possible to override wait_for_all by explicitly setting it to 0. If more than 2 nodes join the cluster, the two_node option is automatically disabled.](https://www.systutorials.com/docs/linux/man/5-votequorum/)

