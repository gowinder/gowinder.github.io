# proxmox ve 部署双节点HA集群及glusterfs分布式文件系统笔记


# proxmox ve 部署双节点HA集群及glusterfs分布式文件系统

**需要保存两个pve节点的版本一样**

两个`proxmox ve`节点,实现高可用`vm`,`lxc`自动迁移

## 修改`hosts`文件

在两台`pve`的`/etc/hosts`中,增加如下`host`

```text
192.168.1.229 pve1.local pve1
192.168.1.230 pve2.local pve2
192.168.1.229 gluster1
192.168.1.230 gluster2
```

## 安装`glusterfs`

以下操作都在两台机器上做, 这里分别为`pve1`, `pve2`

```shell
wget -O - https://download.gluster.org/pub/gluster/glusterfs/9/rsa.pub | apt-key add -

DEBID=$(grep 'VERSION_ID=' /etc/os-release | cut -d '=' -f 2 | tr -d '"')
DEBVER=$(grep 'VERSION=' /etc/os-release | grep -Eo '[a-z]+')
DEBARCH=$(dpkg --print-architecture)
echo deb https://download.gluster.org/pub/gluster/glusterfs/LATEST/Debian/${DEBID}/${DEBARCH}/apt ${DEBVER} main > /etc/apt/sources.list.d/gluster.list

apt update && apt install -y glusterfs-server
```

**需要保存gluster的版本一样**

```shell
gluster --version
```

如果一样,请升级,我有一台是`9.2`,加了源也没不会直接升级到`10.3`,需要这样做:

```shell
apt-get list -a glusterfs-server
```

会出现全部的版本,然后安装对应的

```shell
apt-get install glusterfs-server="10.3-1"
```

在`pve1`上编辑: `nano /etc/glusterfs/glusterd.vol`, 在 `option transport.socket.listen-port 24007` 增加:

```text
    option transport.rdma.bind-address gluster1
    option transport.socket.bind-address gluster1
    option transport.tcp.bind-address gluster1
```

在`pve2`上编辑: `nano /etc/glusterfs/glusterd.vol`, 在 `option transport.socket.listen-port 24007` 增加:

```text
    option transport.rdma.bind-address gluster2
    option transport.socket.bind-address gluster2
    option transport.tcp.bind-address gluster2
```

### 开启服务

```shell
systemctl enable glusterd.service
systemctl start glusterd.service
```

**重要,在`pve2`上需要执行命令以加入集群**

```shell
gluster peer probe gluster1
```

显示: `peer probe: success` 就OK

### 增加`volume`

这里直接建立在`zfs`上,保证两台`pve`都使用一样的路径建立`volume`,这里的路径是`/tank/gfs`
**注意如果是zfs的话,不要能使用挂载根目前,而要使用目录**, 下面例子中,`/tank/gfs`就是一个`zfs`的挂载,在下面建立了一个子目录`s`来用作`volume`

```shell
gluster volume create VMS replica 2 gluster1:/tank/gfs/s gluster2:/tank/gfs/s

gluster vol start VMS
```

### 检查状态

```shell
gluster vol info VMS
gluster vol status VMS
```

### 增加挂载

在两台`pve`上都要做
```shell
mkdir /vms
```

修改`pve1`的`/etc/fstab`,增加

```text
gluster1:VMS /vms glusterfs defaults,_netdev,x-systemd.automount,backupvolfile-server=gluster2 0 0
```

修改`pve2`的`/etc/fstab`,增加

```text
gluster2:VMS /vms glusterfs defaults,_netdev,x-systemd.automount,backupvolfile-server=gluster1 0 0
```

重启两台`pve`,让`/mnt`挂载

#### 两台`pve`不重启挂载

```shell
mount /vms
```

## 解决`split-brain`问题

两个节点的`gluster`会出现`split-brain`问题,就是两节点票数一样,谁也不听谁的,解决办法如下:

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

## `pve`双节点集群设置

第一个创建,第二个加入,没什么好说的

### 创建共享目录

在`DataCenter`中的`Storage`中,点`Add`,`Directory`填`/vms`, **钩选 share**

### HA设置

修改 `/etc/pve/corosync.conf`

在`quorum`中增加,变成这样:

```json
quorum {
    provider: corosync_votequorum
    expected_votes: 1
    two_node: 1
}
```
其中, `expected_votes`表示希望的节点数量, `two_node: 1`表示,只有两个节点,还有一个`wait_for_all: 0`, [NOTES: enabling two_node: 1 automatically enables wait_for_all. It is still possible to override wait_for_all by explicitly setting it to 0. If more than 2 nodes join the cluster, the two_node option is automatically disabled.](https://www.systutorials.com/docs/linux/man/5-votequorum/)

