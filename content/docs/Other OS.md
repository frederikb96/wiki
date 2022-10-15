---
title: "Other OS's"
date: 2022-10-15
draft: false
---

# Other OS

Information about Other OS's

## Windows Partition / Restore

### Restore EFI

https://www.german-syslinux-blog.de/windows-10-efi-partition-reparieren-wiederherstellen/comment-page-2/
https://docs.microsoft.com/de-de/windows-server/administration/windows-commands/diskpart

```

WindowsPE booten and lang us, rest german repair pc trouble

command line start "diskpart"
sel = select
par = partition
vol = volume

list disk
sel disk 0
list par

create partition efi [size=<n>] [offset=<n>] [noerr] # offset in kb
create partition EFI size=100
format quick fs=fat32 label="System" assign letter="z"
exit

bcdboot c:\\windows /l en-us /s z: /f UEFI

exit

```

### Check Disk

After gparted e.g.:

boot WindowsPE and then:

```
chkdsk /?
chkdsk c:
```

### Dual Boot

Ubuntu entry:

```
bcdedit /enum firmware
bcdedit /copy {bootmgr} /d "Ubuntu Windows"
bcdedit /set {guid} path \EFI\ubuntu\shimx64.efi
bcdedit /set {fwbootmgr} displayorder {guid} /addfirst
```

Windows entry:

```
bcdedit /set {fwbootmgr} displayorder {bootmgr} /addfirst
```

Script:

```
ECHO change boot to ubuntu
bcdedit /set {fwbootmgr} displayorder {7ef59265-d42e-11ea-84fa-87bc0e53bae8} /addfirst
PAUSE
shutdown /r /t 0
```

Old method:

```
Bcdedit /set {bootmgr} path \EFI\Microsoft\Boot\bootmgfw.efi
Bcdedit /set {bootmgr} path \EFI\ubuntu\shimx64.efi
```



## FP4 / Android

### Install

- Unlock bootloader: https://support.fairphone.com/hc/en-us/articles/4405858258961
  - SN and IMEI in Sachen note
  - Necessary to keep unlocked if device is updatend an thus magisk removed, then you need unlocked bootloader again to access root
  - **Note**, locking bootloader and also unlocking deletes all data on phone!
- Install /e/ OS: https://doc.e.foundation/devices/FP4/install
  - Has custom recovery with root access

### Root

https://forum.fairphone.com/t/fp4-root-access-is-possible-maybe-a-bit-risky/76839

https://forum.xda-developers.com/t/fairphone-4-root.4376421/

After booting phone works, do following:

- install magisk apk from magisk website
- use boot.img from e os downloaded zip of your installed version to patch that image
- transfer it to your pc
- boot into fastboot
- `adb reboot bootloader`
- `fastboot boot /path/to/patched_boot.img`
- boot will take some time
- then in magisk use direct install
- problems might occur if slot for boot image is changed

[Hide Root:](https://www.reddit.com/r/androidroot/comments/t2y3mq/vr_securego_plus_root_detection/)

- check in settings the zygisk mode and configure deny list
- additional might be necessary to disable magisk temporartily
- Carefull to always enable again before loosing root rights, but root rights are stored if you do not unintall an app, so can keep using without having magisk enabled
- `pm disable com.topjohnwu.magisk pm enable com.topjohnwu.magisk`
- Also see tasker for shortcuts

### Tasker
Shortcuts:
- use tasker to create a task with shell command
- put in the desired command
- Go to tasker app and leave app by pressing back button, Important!
- Then create widget on home screen and select tasker shortcut and select the task, also select an icon
- Done!

### Terminal
Also see tasker, more convenient!
Shortcuts:
- Termux good terminal with root
- Termux shortcuts with add on and place in folder `/data/data/com.termux/files/home/.shortcuts`
- `su - c` for execute as root
- crond install via busybox
- Then create crond dir in e.g `/data/crontab/root`
- Then start with `crond -b -c dir`
- Use for autostart `/data/adb/serives.d` and make script execute as su and also chmod x.


### Obsidian
Use tasker to create launcher
- `obsidian://advanced-uri?vault=Notes&filepath=Eingang.md&mode=append&commandname=Toggle%20keyboard`

### ADB Backup

Data

```
#!/bin/bash

myName="$1"
mySend="$2"

adb -s b3f7e1cb shell tar cf data/data/send.tar -C data/data $myName
adb -s b3f7e1cb pull data/data/send.tar ${mySend}.tar

adb -s ff0983b4 push ${mySend}.tar /data/data/send.tar

USER=$(adb -s ff0983b4 shell stat -c '%U' data/data/$myName)
GROUP=$(adb -s ff0983b4 shell stat -c '%G' data/data/$myName)

adb -s ff0983b4 shell ls -la data/data/$myName

adb -s ff0983b4 shell tar xf data/data/send.tar -C data/data/
adb -s ff0983b4 shell ls -la data/data/$myName

adb -s ff0983b4 shell chown -R ${USER}:${GROUP} data/data/$myName
adb -s ff0983b4 shell chown -R ${USER}:${GROUP}_cache data/data/$myName/c*

adb -s ff0983b4 shell ls -la data/data/$myName
```

Sdcard data

```
#!/bin/bash

myName="$1"
mySend="$2"

adb -s b3f7e1cb shell tar cf sdcard/Android/media/send.tar -C sdcard/Android/media $myName
adb -s b3f7e1cb pull sdcard/Android/media/send.tar ${mySend}.tar

adb -s ff0983b4 push ${mySend}.tar sdcard/Android/media/send.tar

adb -s ff0983b4 shell ls -la sdcard/Android/media

USER=$(adb -s ff0983b4 shell stat -c '%U' data/data/$myName)
GROUP=$(adb -s ff0983b4 shell stat -c '%G' data/data/$myName)

adb -s ff0983b4 shell tar xf sdcard/Android/media/send.tar -C sdcard/Android/media/
adb -s ff0983b4 shell ls -la sdcard/Android/media/$myName

adb -s ff0983b4 shell chown -R ${USER}:everybody sdcard/Android/media/$myName

adb -s ff0983b4 shell ls -la sdcard/Android/media/$myName
```

Old

```
find ./ -type d -name "*t*"

tar cf send.tar net.osmand.plus

adb -s b3f7e1cb pull data/data/send.tar tmp/ && adb -s ff0983b4 push tmp/send.tar /data/data/

ls -l net.osmand.srtmPlugin.paid
tar xf osmandSrtm.tar && ls -l net.osmand.srtmPlugin.paid
chown -R u0_a233:u0_a233 net.osmand.srtmPlugin.paid
chown -R u0_a233:u0_a233_cache net.osmand.srtmPlugin.paid/c* && ls -l net.osmand.srtmPlugin.paid



adb -s b3f7e1cb shell find /data/data -type d -name "*whatsapp*"
```

### OS Update with root

- Handy download new zip from e os website from favoriten
- extract boot and patch boot file
- enable Magisk
- update fp4 auto updates
- `adb reboot bootloader`
- `fastboot boot /home/freddy/magisk*.img`
- Magisk install root