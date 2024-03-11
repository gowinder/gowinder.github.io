# 直接使用cifs.ko挂载WINDOWS 共享


在网上找到的,[http://myapad.wordpress.com/author/xaueious/](http://myapad.wordpress.com/author/xaueious/ "http://myapad.wordpress.com/author/xaueious/")

这个家伙,用爱可视的内核源代码编了一个cifs的RK2808可用模块

把这个下载下来,放在flash随便什么目录下面,假设是/flash/cifs.ko

然后在命令行执行

insmod /flash/cifs.ko

将模块挂载,之后就可以挂载网络共享了

mount –t cifs –o username=guest,iocharset=utf8 //192.168.0.3/movie /sdcard/movie

username我指定的是guest用户,所以没有密码,如果有密码,还要加一个 password=somekey,

//192.168.0.3/movie是WINDOWS共享的位置, /sdcard/movie 是本地挂载目录

注意一定要指定iocharset=utf8 不然中文就是乱码,OK,这比使用NFS还要方便,不用在台式机上装NFS服务器,也不会有乱码.

cifs.ko我已经上传到群共享了.

QQ群:83895562
