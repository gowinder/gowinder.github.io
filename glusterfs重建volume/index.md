# glusterfs重建volume


# glusterfs重建volume

## 启因

因为之前做`glusterfs`集群时,比较急,直接用的`pve`主机的网络地址,运行了几天,发现`volume`同步的时候,占用了`pve`本身的网络通道,正好`dell r720`有4个网口,`e3`主机上有2个,所以都拿出一个来单独做`gluster`的绑定地址

结果没想到,我直接在两边的`hosts`文件里面将对应`gluster1`, `gluster2`的`ip`修改了,结果不行,还是走的原来的地址,而且,`gluster1`上一直显示`gluster`不在线,于是我就想删除了`volume`重新配置.

## 过程

先用`new bing`问了一下他,怎么删除`volume`, 他给出了步骤,不过我发现有一步反了,我自己试了一下,按如下步骤:

- 两边都执行:停止`volume`: `sudo gluster vol stop VMS`
- 两边都执行:`sudo umount gluster1:/tank/gfs/s`
- 两边都删除`brick`: `sudo gluster vol remove-brick VMS replica 1 gluster2:/tank/gfs/s force`, 注意,`gluster2`删除`gluster1`,`gluster1`删除`gluster2`
- `sudo gluster vol delete VMS start`
- 这一步会报错,说有`peer`还在,因为我在`gluster1`上显示`gluster2`一直是`disconnect`,因此要删除`peer`
- 查看`peer`: `sudo gluster peer statue`, 会显示是`disconnected`
- 两边都删除`peer`: `sudo gluster peer detach gluster2`
- 再次`sudo gluster peer status`显示已经没有`peer`了
- 删除`volume`: `sudo gluster volume delete VMS` 显示删除成功

### 重新加入时,如果报错

两边节点都要做:

- `sudo systemctl stop glusterd`

- `peer probe: failed: g1 is either already part of another cluster or having volumes configured`, 就需要在`g1`的主机上,将`/var/lib/glusterd`目录下面除`glusterd.info`文件的其它全部删除掉,再去`sudo glusterd peer probe g1`

- 删除`/tank/gfs/s/.glusterfs`

但是还是会报错`/tanks/gfs/s`已经在一个`volume`中了,没办法我又重新建了一个目录

**重要**, 新建的目录,一定要`mount`之后,再向`mount`之后的目录里面拷贝,如:`sudo mount g1:vms`, 挂载到`/vms`之后,向`/vms`里面拷贝,不然无法同步,参见: [stackoverfow](https://stackoverflow.com/a/29301051)

## 总结

`gluster`节点的地址不能随便改,需要用命令去设置

### 正确的做法 (未试过)

- 停止`volume`: `sudo gluster volume stop VMS`
- `sudo umount /vms`
- `gluster volume set VMS config.transport tcp`
  
- ```shell
  gluster volume set VMS config.transport.tcp.bind-address newaddress
  gluster volume set VMS config.transport.socket.bind-address newaddress
  gluster volume set VMS config.transport.rdma.bind-address newaddress
  ```

- `sudo gluster volume start VMS`
- `sudo mount /vms`

