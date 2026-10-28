# Drastic Disk Performance Drop Caused by NVMe Power Management in Proxmox VE


# Drastic Disk Performance Drop Caused by NVMe Power Management in Proxmox VE

A few days ago I deployed a two-node `proxmox ve` cluster. It worked fine at first, but after a day, when I logged in to the `dell r720` host over `ssh`, I found the `shell` responding very slowly, and going into the `linux container` on it was even worse — every time I typed a command, `zsh` would freeze for quite a while.

I thought it was a problem with `zsh`, so I switched to `bash` and `fish`, but they were only slightly better. Opening a local file with `vim` was also very slow — could the disk be the problem? The system disk on this `dell` host is a `West Digital` `SN750 1T`, which by all accounts shouldn't be slow, so I ran a test:

```shell
time dd if=/dev/zero of=tst.out bs=4M count=256
```

I had no idea until I tested it, and the result gave me a shock:

```shell
256+0 records in
256+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 42.666 s, 24 MB/s
```

That was abnormally slow. Neither `sudo dmesg | grep -i error` nor `sudo grep -i 'error' /var/log/syslog` revealed any problem. It is the system disk and hadn't been touched since installation.

I tried the good old reboot, and found that booting into the system took a very long time. Checking with `smarttools` showed no bad sectors and no errors at all. I was frustrated. After searching on `google` and `bing` for a long time without much to show, one answer mentioned in passing that it might be an `apm` power management issue and that `hdparm` should be used to inspect the settings. I looked for a long time, but it had no effect on my disk and just threw errors.

After thinking for a while, I realized I had been searching for `ssd`, so I switched the keyword to `nvme` and searched again.

I found a new tool: `nvme-ctl`. Someone said to use it to check the `nvme` power management configuration. I checked its `manpage` and saw that `nvme get-feature` is used to inspect power management, but it didn't say what the `--feature-id` parameter should be, so I kept looking.

On `nvme`'s [official site](https://nvmexpress.org/resource/technology-power-features/) I found one: `feature-id`=`0x02` refers to power management. I tried running it:

```shell
sudo nvme get-feature -f 0x02  /dev/nvme0 -H
```

And got the result:

```shell
get-feature:0x2 (Power Management), Current value:0x000003
        Workload Hint (WH): 0 - No Workload
        Power State   (PS): 3
```

There was a result, but what exactly is `0x03`? I remembered that when I checked earlier with `smarttools` there seemed to be a list of power states, so I tried it, and sure enough there was one:

```shell
Supported Power States
St Op     Max   Active     Idle   RL RT WL WT  Ent_Lat  Ex_Lat
 0 +     3.50W    2.90W       -    0  0  0  0        0       0
 1 +     2.70W    1.80W       -    0  0  0  0        0       0
 2 +     1.90W    1.50W       -    0  0  0  0        0       0
 3 -   0.0200W       -        -    3  3  3  3     3900   11000
 4 -   0.0050W       -        -    4  4  4  4     5000   39000
```

As you can see, `0x03` is drastically different from the previous ones — it's basically asleep. I set it to `0x02` to try:

```shell
sudo nvme set-feature /dev/nvme0 -f 0x02 --value=2
```

The result was satisfying:

```shell
4096+0 records in
4096+0 records out
16777216 bytes (17 MB, 16 MiB) copied, 0.0757074 s, 222 MB/s
dd if=/dev/zero of=tst.out bs=4k count=4096  0.00s user 0.03s system 39% cpu 0.080 total
```

`222 MB/s` — finally it felt like real read/write speed. I changed it to `0x01`:

```shell
sudo nvme set-feature /dev/nvme0 -f 0x02 --value=0x01
```

Result:

```shell
256+0 records in
256+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 2.3109 s, 465 MB/s
dd if=/dev/zero of=tst.out bs=4M count=256  0.00s user 1.43s system 57% cpu 2.495 total
```

After struggling with this all evening and about to reinstall the system, I found the answer by talking step by step with `new bing`'s `chat gpt`. As capable as `chat gpt` is, asking the right question matters too — it doesn't have the human knack for guessing.

At last, `speed freedom` achieved.

