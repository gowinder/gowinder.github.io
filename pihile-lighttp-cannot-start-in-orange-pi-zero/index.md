# 在Orange Pi Zero上安装pihole后,webui无法启动问题


# 在Orange Pi Zero上安装pihole后,webui无法启动问题

## 启因

最新在空闲`Orange Pi Zero`上安装了`Pihole`用来做家庭主`DNS`服务器,安装一路顺序: `Armbian`->`Pihole`,结果安装好后,无法启动`WebUI`, `Pihole`用的是`Lighttp`

## 解决过程

使用命令检查: `sudo lighttpd -t -f /etc/lighttpd/lighttpd.conf`
结果是: `Syntax OK` 没有问题.

查看`lighttpd`服务: `sudo systemctl status lighttpd`, 发现一启动就退出了,`exited=255`.

继续查看日志: `sudo tail -f /var/log/lighttpd/error-pihole.log` 发现报错:

```shell
2023-02-26 21:53:03: server.c.1513) server started (lighttpd/1.4.59)
2023-02-26 21:53:03: mod_accesslog.c.612) opening log '/var/log/lighttpd/access.log' failed: Permission denied
```

说 `/var/log/lighttpd/` 这个目录没有权限, 于是做下面的操作:

```shell
sudo chown pihole:pihole /var/log/lighttpd -R
sudo chmod 755 /var/log/lighttpd -R
```

再次启动:

```shell
sudo systemctl start lighttpd
```

就ok了.

