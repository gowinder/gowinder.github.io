# proxmox ve 中nvme电源管理问题导致的磁盘性能急剧下降问题


# proxmox ve 中nvme电源管理问题导致的磁盘性能急剧下降问题

这几天部署了`proxmox ve`双节点集群,正常用没什么问题,但是过了一天,`dell r720`这台主机,`ssh`上去发现`shell`的反应好慢,然后进去里面的`linux container`,也是反应慢得狠,每打一个命令,`zsh`都要卡半天.

以为是`zsh`的问题,换了`bash`, `fish`只是强一点点,用`vim`打开本地文件也是很慢,是不是硬盘出了问题? 这台`dell`主机上的系统盘是`West Digital`的`SN750 1T`,照说应该不慢,于是我测了一下:

```shell
time dd if=/dev/zero of=tst.out bs=4M count=256
```

不测不知道,一测吓一跳:

```shell
256+0 records in
256+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 42.666 s, 24 MB/s
```

慢得不正常, `sudo dmesg | grep -i error`, `sudo grep -i 'error' /var/log/syslog` 都没发现有什么问题,是系统磁盘,装好后就没动过.

重启大法试一下,发现进系统需要好长时间,用`smarttools`,查看,也没有坏道,没任何错误,郁闷了,`google`,`bing`搜索了半天,没有太大收获,有一个回答里面带过可能是`apm`电源管理的问题,要使用`hdparm`来查看设置,找了半天,对我的硬盘没效果,会报错.

想了半天,发现我搜索的是`ssd`,我换了个关键字`nvme`去搜索.

发现了一个新工具: `nvme-ctl`, 有人说用这个来检查`nvme`的电源管理配置,查了下`manpage`,要用`nvme get-feature`来查看电源管理,但是里面的参数`--feature-id`是什么没说,又继续找.

在`nvme`的[官网](https://nvmexpress.org/resource/technology-power-features/)找到一个,`feature-id`=`0x02`是指电源管理,试着运行了一下:

```shell
sudo nvme get-feature -f 0x02  /dev/nvme0 -H
```

得到结果:

```shell
get-feature:0x2 (Power Management), Current value:0x000003
        Workload Hint (WH): 0 - No Workload
        Power State   (PS): 3
```

有结果了,可是`0x03`具体是多少呢?我想起来之前用`smarttools`查看的时候好像有电源配置列表,试了一下,真有:

```shell
Supported Power States
St Op     Max   Active     Idle   RL RT WL WT  Ent_Lat  Ex_Lat
 0 +     3.50W    2.90W       -    0  0  0  0        0       0
 1 +     2.70W    1.80W       -    0  0  0  0        0       0
 2 +     1.90W    1.50W       -    0  0  0  0        0       0
 3 -   0.0200W       -        -    3  3  3  3     3900   11000
 4 -   0.0050W       -        -    4  4  4  4     5000   39000
```

可以看到`0x03`和前面的差距好大,基本就是休眠了,把他调到`0x02`试试:

```shell
sudo nvme set-feature /dev/nvme0 -f 0x02 --value=2
```

结果舒服了:

```shell
4096+0 records in
4096+0 records out
16777216 bytes (17 MB, 16 MiB) copied, 0.0757074 s, 222 MB/s
dd if=/dev/zero of=tst.out bs=4k count=4096  0.00s user 0.03s system 39% cpu 0.080 total
```

`222 MB/s`读写终于感觉了,给他改到`0x01`:

```shell
sudo nvme set-feature /dev/nvme0 -f 0x02 --value=0x01
```

结果:

```shell
256+0 records in
256+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 2.3109 s, 465 MB/s
dd if=/dev/zero of=tst.out bs=4M count=256  0.00s user 1.43s system 57% cpu 2.495 total
```

折腾了一晚上,准备重装系统了,用`new bing`的`chat gpt`一步步对话找到了答案,`chat gpt`虽然厉害,但是提问也很重要,他可没有人类那么强的猜测能力.

终于实现`速度自由`了.

