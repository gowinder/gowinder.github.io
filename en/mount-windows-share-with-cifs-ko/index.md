# Mounting a Windows Share Directly with cifs.ko


Found this online, [http://myapad.wordpress.com/author/xaueious/](http://myapad.wordpress.com/author/xaueious/ "http://myapad.wordpress.com/author/xaueious/")

This guy compiled a cifs module for the RK2808 using Archos's kernel source code

Download it and put it in any directory on the flash, say /flash/cifs.ko

Then run this from the command line

insmod /flash/cifs.ko

That loads the module, and after that you can mount network shares

mount –t cifs –o username=guest,iocharset=utf8 //192.168.0.3/movie /sdcard/movie

I specified the guest user for username, so there's no password; if there is one, you also need to add password=somekey,

//192.168.0.3/movie is the location of the Windows share, /sdcard/movie is the local mount point

Note that you must specify iocharset=utf8, otherwise Chinese text comes out as garbage. OK, this is even handier than NFS — no need to install an NFS server on the desktop machine, and no garbled characters either.

I've already uploaded cifs.ko to the group share.

QQ group: 83895562

