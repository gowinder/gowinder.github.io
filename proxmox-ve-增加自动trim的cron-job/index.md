# proxmox ve 增加自动trim的cron job


# proxmox ve 增加自动trim的cron job

## 检查状态

```shell
sudo systemctl status fstrim.timer
# output:
● fstrim.timer - Discard unused blocks once a week
     Loaded: loaded (/lib/systemd/system/fstrim.timer; disabled; vendor preset: enabled)
     Active: inactive (dead)
    Trigger: n/a
   Triggers: ● fstrim.service
       Docs: man:fstrim
```

## 开启服务

```shell
sudo systemctl start fstrim.timer
```

## 检查下次清理时间

```shell
sudo systemctl list-timers --all
# output:

NEXT                        LEFT           LAST                        PASSED     UNIT                         ACTIVATES
Fri 2023-02-17 23:54:54 CST 8h left        Thu 2023-02-16 23:54:54 CST 15h ago    systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.servi>
Sat 2023-02-18 00:00:00 CST 8h left        Fri 2023-02-17 00:00:00 CST 15h ago    atop-rotate.timer            atop-rotate.service
Sat 2023-02-18 00:00:00 CST 8h left        Fri 2023-02-17 00:00:00 CST 15h ago    logrotate.timer              logrotate.service
Sat 2023-02-18 00:00:00 CST 8h left        Fri 2023-02-17 00:00:00 CST 15h ago    man-db.timer                 man-db.service
Sat 2023-02-18 04:19:48 CST 13h left       Fri 2023-02-17 05:29:22 CST 9h ago     pve-daily-update.timer       pve-daily-update.service
Sat 2023-02-18 05:18:55 CST 14h left       Fri 2023-02-17 06:45:48 CST 8h ago     apt-daily.timer              apt-daily.service
Sat 2023-02-18 06:41:05 CST 15h left       Fri 2023-02-17 06:02:36 CST 9h ago     apt-daily-upgrade.timer      apt-daily-upgrade.service
Sun 2023-02-19 03:10:12 CST 1 day 12h left Sun 2023-02-12 03:10:19 CST 5 days ago e2scrub_all.timer            e2scrub_all.service
Mon 2023-02-20 00:37:59 CST 2 days left    n/a                         n/a        fstrim.timer                 fstrim.service
n/a                         n/a            n/a                         n/a        pvesr.timer
```

完成

