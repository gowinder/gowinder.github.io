# APAD RK2808: Notes on Getting Root, and Mounting an NFS Network Drive


See: [Getting root on the RK2808 System disk image, plus material on other custom firmware builds](http://hiapad.com/bbs/forum.php?mod=viewthread&tid=47&extra=page%3D2)

[A basic tutorial for modifying Rockchip RK28XX Android system firmware images](http://lockeyue.spaces.live.com/blog/cns!B3988DF30734BED3!1553.entry)

It was a real pain. I did it many times, and I found that tutorials exist but none of them are complete — miss one step and you may end up without root.

Preparation:

A flashing tool. Download link: [flashing tool](http://cid-b3988df30734bed3.office.live.com/self.aspx/.Public/%E7%BD%91%E7%BB%9C/%E7%91%9E%E8%8A%AF%E5%BE%AERK2808%5E_android%E7%B3%BB%E7%BB%9F%E5%9B%BA%E4%BB%B6%E4%BF%AE%E6%94%B9%E5%B7%A5%E5%85%B7%E5%8C%85%5E_Ver01.rar)

A ROM. There are lots of versions, some stock, some modified. I used a modified one, hybrid 5.1, downloaded from a foreign forum — [hybrid 5.1](http://www.slatedroid.com/apad-rogerbraun-firmware/566-%5B-piece-software-flashed-device-expand-its-functionality%5D-hybrid-custom-firmware-apad-5-1-batch-2-red-led-only-july-3rd.html). Note that this ROM claims to have root enabled, but in fact it does not. You can join the QQ group, the shared files are in there: 83895562

A Linux system. If you don't have one, just install a virtual machine — VirtualBox is recommended — and throw Ubuntu on it, and you're good to go.

You need the cramfs tools installed, because the goddamn RK2808 uses that filesystem. Google it or BING it for the details; put simply, it is a compressed, read-only image, so you have to modify the firmware before flashing.

Unpack update.img and copy it under UBUNTU. Let's assume you copied it to /home/zzz/rom/ (note: zzz is your login user name)

Then run

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
This step unpacks the IMG into a system directory. If you are using the 5.1 firmware, su is already in there and you only need to fix the permissions; if not, go download one: [SUPERUSER from XDA](http://forum.xda-developers.com/showthread.php?t=682828)
``````
Then copy the system/bin/su inside it to system/bin/su under the directory you just unpacked, and to system/xbin/su — note you have to copy both.
``````
Copy system/app/Superuser.apk to the corresponding directory too, mind the upper and lower case.
``````
Then run
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

This step is about the permissions. I failed N times in the past, and the thing I was missing was those first two lines — you absolutely have to give ownership of these files to the root user.

OK, once the above is done you can pack SYSTEM.IMG again.

```
mkcramfs  system system.img 

```

  
Copy this system.img into the image directory you unpacked earlier, then use the tool to repack it into udpate.img and flash it.

Install BetterTerminal, which is just a terminal app. Once it's installed, type su in the terminal and hit enter; you'll get a prompt, choose yes, and if the prompt changes from $ to # that means it worked. If it says Permission denied, it didn't.

  
  

The following is the work needed for the NFS network drive. If you're not planning to mount NFS, you can skip it.

See: [CIFS and NFS modules](http://www.slatedroid.com/nationite-midnite-firmware-development/8087-cifs-nfs-modules-3.html)

[Windows NFS configuration](http://ygliang.blog.51cto.com/69909/52898)

Install [Windows Services for UNIX Version 3.5](http://www.microsoft.com/downloads/en/details.aspx?FamilyID=896c9688-601b-44f1-81a4-02878ff11778) on the desktop machine you're sharing from. You're all on WINDOWS, right? I only tried XP; never tried WIN7, but it should be much the same. If it's LINUX you can probably just turn it on directly.

The main point is that Android 1.5 doesn't support the CJFS format — the kernel doesn't support it — and I can't compile a kernel (I'm learning), otherwise I could just copy an SMB share (that is, a WINDOWS share) into a local directory, and then novels, movies and so on

wouldn't have to be copied from the desktop to the tablet — I could reach them any time over the share and WiFi, and just play the movie directly. Sadly CJFS isn't supported, so there was another way, and it took me ages to find it: the NFS format, which I stumbled on by accident on a foreign forum.

The 1.5 kernel supports NFS.

First download [Windows Services for UNIX Version 3.5](http://www.microsoft.com/downloads/en/details.aspx?FamilyID=896c9688-601b-44f1-81a4-02878ff11778), then configure the share following the reference post above. Note that in the local security policy there is one item you absolutely must enable, otherwise you can mount it but the directory comes up empty:

Network access: Let Everyone permissions apply to anonymous users

The way to get there: run gpedit.msc, under "Computer Configuration - WINDOWS Settings - Security Settings - Local Policies - Security Options".

Then in the [CIFS and NFS modules](http://www.slatedroid.com/nationite-midnite-firmware-development/8087-cifs-nfs-modules-3.html) thread there is a MountNfs.apk — download and install it, then run MOUNTNFS. In the first dialog box fill in the address; the format should be: IP:/FOLDER\_NAME, where IP is your WINDOWS IP and FOLDER\_NAME is the name you set on the NFS share (note: not the target name on your hard disk, but the NFS share name in the properties). Don't leave out the colon.

Below that, fill in the directory on the tablet — create an empty one yourself first, such as sdcard/movie — then tap MOUNT. If it says the mount succeeded, you're done: browse that directory with ES File Explorer and you'll see the share. (If you don't see it, use a terminal, switch to su, and check with ls: ls /sdcard/movie. If it's there, it's most likely that the WINDOWS policy wasn't changed — that's exactly what happened to me: the mount succeeded, but the file manager couldn't see it, while ls as the superuser could.)

Click a movie and it plays straight away, no need to copy anything over. Just make sure the WiFi signal is good, otherwise it will stutter.

One problem I haven't solved: NFS doesn't support UTF8, so Chinese text comes out as garbage. Best to rename the files to English by hand, haha.

If you have any questions, come and talk in the QQ group.

Also, my GOOGLE MAP can't get a fix on my location — turning on "My Location" in the settings doesn't help either. Anyone know why?

