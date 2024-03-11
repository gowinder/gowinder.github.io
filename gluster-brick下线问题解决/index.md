# gluster brick下线问题解决


# gluster brick下线问题解决

## 下线问题

刚装好`gluster`,在上面存放了`linux container`的`rootfs`,这样`linux container`就可以在两台`pve`的群集中做高可用了.

空闲的时候,用命令:`sudo gluster vol status VMS`查看了下,发现有一个`brick`是`offline`的状态.

网上查了下,安装`gstatus`:

```shell
curl -fsSL https://github.com/gluster/gstatus/releases/latest/download/install.sh | sudo bash -x
gstatus --version
```

安装完成后, 运行: `sudo gstatus -a -b`, 发现: `gluster1`是`offline`状态, 查了下,找到一个解决办法: [强制启动](https://github.com/gluster/glusterfs/issues/2742#issuecomment-908361146), 照着来:

```shell
sudo gluster vol start VMS force
```

运行后不一会儿就正常了.

## heal问题

查看`gluster`的时候,发现需要`heal`,

- 查看: `sudo gluster heal VMS info` 查看是否有需要`heal`的
- `heal`: `sudo gluster heal VMS`,
- 查看汇总: `sudo gluster vol heal VMS summary`
- 查看统计: `sudo gluster vol heal VMS statistics heal-count`

