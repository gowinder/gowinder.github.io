# APAD RK2808 开ROOT权限注意事项,挂NFS网络硬盘


参见:[RK2808 System磁盘映像拿到root权限, 其他自作自定义固件的资料](http://hiapad.com/bbs/forum.php?mod=viewthread&tid=47&extra=page%3D2)

[瑞芯微RK28XX\_安卓(android)系统固件包修改基础教程](http://lockeyue.spaces.live.com/blog/cns!B3988DF30734BED3!1553.entry)

很麻烦,搞了很多次,发现教程有,但是都不完全,差一步,就可能开不了ROOT.

准备工作:

刷机工具.下载地址:[刷机工具](http://cid-b3988df30734bed3.office.live.com/self.aspx/.Public/%E7%BD%91%E7%BB%9C/%E7%91%9E%E8%8A%AF%E5%BE%AERK2808%5E_android%E7%B3%BB%E7%BB%9F%E5%9B%BA%E4%BB%B6%E4%BF%AE%E6%94%B9%E5%B7%A5%E5%85%B7%E5%8C%85%5E_Ver01.rar)

ROM,有很多版本,有原始的,有修改版的,我用的修改版的,从国外论坛上下载的[hybrid 5.1](http://www.slatedroid.com/apad-rogerbraun-firmware/566-%5B-piece-software-flashed-device-expand-its-functionality%5D-hybrid-custom-firmware-apad-5-1-batch-2-red-led-only-july-3rd.html) 注意此ROM号称开了ROOT,实际上是没有的.可以加QQ群,里面的共享有:83895562

LINUX系统,没有LINUX系统就装个虚拟机啦,推荐VirtualBox,然后安装一个Ubuntu,就很方便了.

要安装cramfs工具,因为天杀的RK2808使用的就是这个系统,具体的请谷歌或者BING,简单的说就是一个压缩包,是只读的系统,因此必须要在刷机前对固件进行修改.

装update.img解开后,拷贝到UBUNTU下面.这里假设拷贝到了/home/zzz/rom/ 下面(注:zzz是你的登录用户名)

然后运行

```
cd /home/zzz/rom    
mkdir system
sudo mount -t cramfs -o loop system.img  system
tar cvzf system.tgz system
sudo umount system
rm -r system
tar xzvf system.tgz
```

```
这一步就是将IMG解压到一个system目录中,如果你用的是5.1的那个固件,里面已经有SU了,只需要改一下权限就可以了,如果没有,就去下载一个:[XDA的SUPERUSER](http://forum.xda-developers.com/showthread.php?t=682828)
``````
然后将里面的system/bin/su 拷贝到刚才解开目录下面的system/bin/su, system/xbin/su 注意两个都要拷
``````
system/app/Superuser.apk 也拷到对应目录下去,注意大小写.
``````
然后运行
```

```
```
sudo chown root system/bin/su
sudo chown root system/xbin/su
sudo chmod 4577 system/bin/su
sudo chmod 4577 system/xbin/su
sudo chmod 644 system/app/Superuser.apk
```
``````
  

```

这一步就是给权限,我以前搞了N次失败了,就是差前面两句,一定要将此文件的所有权分配给root用户才行.

好了,上面完成后,就可以打包SYSTEM.IMG了

```
mkcramfs  system system.img 

```

  
将这个system.img拷贝到原来解开的Image目录去,然后用工具重新打包成udpate.img,刷机

安装一个BetterTerminal,就是一个终端工具,装好后在终端输入su回车,会有一个提示,选yes,然后终端上的提示符由$变为#就表明成功了,如果显示Permission denied就是没成功.

  
 

以下是开NFS网络硬盘所需要的工作,如果你不准备挂NFS,可以跳过

参见:[CIFS and NFS modules](http://www.slatedroid.com/nationite-midnite-firmware-development/8087-cifs-nfs-modules-3.html)

[windows NFS 配置](http://ygliang.blog.51cto.com/69909/52898)

在共享的台式机上安装[Windows Services for UNIX Version 3.5](http://www.microsoft.com/downloads/en/details.aspx?FamilyID=896c9688-601b-44f1-81a4-02878ff11778),应该都是WINDOWS系统吧,只试过XP,WIN7没试过,应该差不多.如果是LINUX好像就可以直接开了

主要目的,就是因为1.5的ANDROID不支持CJFS格式,是内核不支持,咱又不会编内核(现在在学习中),不然就可以直接拷SMB共享(就是WINDOWS的共享啦)到本地目录,然后什么小说啊,电影啊,之类 的,就

不用从台式机上拷贝到平板,而可以直接通过共享,WIFI来随时访问了,直接播放电影什么的.可惜不支持CJFS格式,所以又有一种方式,搞了好半天才找到,就是用NFS格式,在国外的论坛上无意中看到的,

1.5的内核支持NFS

先下载[Windows Services for UNIX Version 3.5](http://www.microsoft.com/downloads/en/details.aspx?FamilyID=896c9688-601b-44f1-81a4-02878ff11778),然后按照上面的参考帖子配置共享,注意在本地安全策划中,有一项一定要打开,不然可以挂载,但是目录却是空的:

网络访问: 将 Everyone 权限应用于匿名用户

具体方法,运行gpedit.msc,在"计算机配置-WINDOWS设置-安全设置-本地策略-安全选项"下面.

然后在[CIFS and NFS modules](http://www.slatedroid.com/nationite-midnite-firmware-development/8087-cifs-nfs-modules-3.html)这个帖子中,有一个MountNfs.apk,下载安装,然后运行MOUNTNFS,在第一个对话框上填写地址,格式应该是: IP:/FOLDER\_NAME,IP就是你的WINDOWS的IP,FOLDER\_NAME就是在NFS共享上设置的名字(注意不是你硬盘上的目标名,而是属性里面NFS 共享的名字),注意不要掉了:号.

下面填写平板上的目录,自己先建一个空目录,如sdcard/movie,然后点MOUNT,如果提示MOUNT成功,就OK了,用ES文件管理器查看此目录,就可以看到共享了(如果没有,使用终端,然后切换到su,使用ls命令查看:ls /sdcard/movie 看看有没有,如果 有,很可能是WINDOWS的策略没改,我之前就是这样,MOUNT成功了,但是用文件管理器看不到,用超级用户用LS命令可以看到)

点击一个电影,就可以直接播放了,不用拷过来,注意WIFI要信号好才行,不然就会卡.

有一个问题还没解决,就是NFS不支持UTF8,所以中文是乱码,最好手动把文件名改成英文,哈哈

有什么问题可以到QQ群交流.

还有我的GOOGLE MAP无法定位自己的位置,在设置里面开了"我的位置"也没用,哪位知道
