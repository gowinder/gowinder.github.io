# Migrating My Home Server from a Dell R720 to a Snail Interstellar + MacBook Pro


# Migrating My Home Server from a Dell R720 to a Snail Interstellar + MacBook Pro

A few years ago I got into `HomeLab` and bought a second-hand `Dell R720` rack server to use as a home server. But this server's power consumption was too high—dual `cpu`s idled at nearly 200w. After switching to low-power `Xeon`s and installing only one `cpu`, it was still 100w+. Over a year the electricity bill added up, and in summer it got very hot. Later I used the `ipmi` tool and wrote some code in `python` to adjust the loud fans based on temperature, which made it much more comfortable, but in summer as the room temperature rose the fans still ramped up their speed dramatically. I always wanted to switch to a low-power server. I'd previously bought a `Snail Interstellar`, which uses a `j1900` and has very low power consumption, so I migrated the home server from the `Dell R720` to the `Snail Interstellar`.

## Attempt

After bringing the `Dell R720` home, I installed way too many `vm`s and `lxc`s over time, tinkered with `k8s`, `rancher`, and so on. I upgraded the memory to `64G`, and could have added more—after all, `ddr3 ecc` server memory is so cheap—but `64g` is plenty and I can't even use it all; more would just waste power.

Currently I've kept two `lxc`s. One runs `bt` and `qt`—I use `qbittorrent` for both, keeping the `bt` and `pt` services separate.
So far I've downloaded nearly `40t` of data with `qbittorrent`:

![alt text](qbittorrent-statistics-1.png)

`qbittorrent` is very stable. I used `transmission` before and found it unpleasant to use, with mediocre speed. This `lxc` absolutely has to be migrated.

The other `lxc` also runs `ubuntu` and hosts the entire media workflow:

- `jellyseerr` for searching for movies
- `prowlarr` for searching `index`ers and submitting download tasks
- `sonarr` for submitting `TV series` tasks
- `radarr` for submitting `movie` tasks
- `jellyfin` for playing videos

This workflow has become very stable and needs to be migrated. Since these don't use much in the way of resources, leaving them on the `R720` wastes too much.

## Steps

- Install `pve` on the `Snail Interstellar`; this is required.
- Configure two `ubuntu` `lxc`s on the `Snail Interstellar`; this is required.
- Use `rsync` to sync over all the `docker-compose` files and the mounted `config` folders that the various `docker` containers used.
- On the `Snail`, use `mount -t nfs` to mount the media resources and download directories.
- Change the directories mapped in `docker-compose` to the mounted directories.
- Run `docker-compose up -d` to start all services.

And that was it. Why mount via `nfs` instead of just moving the drives over directly? Because I didn't know whether the `Snail`'s `j1900` + `4G` of memory could handle all these services. I tested first to confirm it could, and would migrate the drives at the end.

## Problems Found

### Insufficient Memory

Shortly after the `Snail Interstellar` boots, `cpu` usage jumps to 100%, but checking with `top`, it's a process named `kswapd`. I googled it—it's the `swap space` management process, which means `4g` of memory isn't enough.

By default `swap` is only `512M`, so I planned to use `swapon` and similar commands to create a few GB of `swap` partition, only to find there wasn't enough room on the `root` partition. Because I was just testing at first, I'd connected a `120G` `ssd` to the built-in `sata` on the `Snail` to serve as the `root` partition, but a `120G` `ssd` is too small—the `lxc` disks alone take up `80G`. I haven't gone out to buy a bigger `sata` `ssd` yet, so I'll make do for now, and later switch directly to a big `ssd` using `clonezilla`.

First I moved the `lxc` disks over to the `zpool` of the mechanical drive array. Being a bit slow doesn't really matter—these are all background services that don't need very fast `io` reads and writes. Then I added `4G` of space with `swapon`, and it's much better now:
![alt text](j1900-htop.png)
The biggest `cpu` consumer is the `baidupan` service, which actually wasn't downloading anything at the time of the screenshot.
As you can see, `qbittorrent` really does use a lot of memory, because there are too many tasks.
![alt text](qbittorrent-status.png)
Hundreds of them—how could it not be a lot.

### `j1900` Decoding Capability

The `j1900` is nearly 10 years old after all, and many decoders aren't supported. Compare decoding with the `n100`:
![alt text](j1900-n100-decode.png)
Especially since the videos downloaded now are all `4k HDR`, mostly encoded in `h.265`, `HEVC`, none of which is supported, so transcoding can't be done, and many clients using `jellyfin` can't play `4k` videos.

So I came up with an idea: isn't my `MacbookPro` at home? The `M1Max` should be able to decode most of them:
![alt text](m1max-n100-decode.png)
Except for `AV1`, basically everything works, so I installed `jellyfin` on the `MacbookPro`. At this point two more problems came up:

1. On the `m1 mac`, after running for a while `jellyfin`'s server side would freeze, for unknown reasons. I searched `issue`s on `github` and others had reported the same problem, but there was no solution.
2. For the `MacbookPro` I use an `HP G2` Thunderbolt `hub` with a wired network adapter connected. Whenever I `close the lid` or `lock the screen`, the wired adapter disconnects. I'd noticed this problem before when using `mac`'s `screen sharing`—connecting via the wired adapter's `ip` doesn't work; but if at that point I connect once via the wireless adapter's `ip`, the wired adapter's `ip` starts working.

#### Solutions

1. Switch to `emby`; no problems.
2. It should be a sleep issue. Using `sudo pmset -g`:

```
System-wide power settings:
 SleepDisabled          0
Currently in use:
 standby              1
 Sleep On Power Button 1
 hibernatefile        /var/vm/sleepimage
 powernap             1
 networkoversleep     0
 disksleep            10
 sleep                0 (sleep prevented by EmbyServer, screensharingd)
 hibernatemode        3
 ttyskeepawake        1
 displaysleep         3
 tcpkeepalive         1
 powermode            0
 womp                 1
```

I used the command `pmset -a disablesleep 1` to disable system sleep, which solved it perfectly. Also, the `m1`'s idle power consumption is really low—a world of difference from the `R720`.

## Other Things to Migrate

- `icloud_photo_backup`: used to back up `icloud` photos
- `cron`: some scheduled tasks, such as renewing certificates with `acme`.
- `cups`: the printer service. I migrated it directly and configured it, but printing didn't respond; in the end I restarted the printer and it worked.

After everything is done, move the drives over and remount them in `/etc/fstab`, and that's it.

