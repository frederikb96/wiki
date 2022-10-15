---
title: "Linux Backup"
date: 2022-10-15
draft: false
---

# Linux Backup

## Clonezilla Backups
You can use clonezilla to make full backups of your complete drive or the Raspberry SD card
https://clonezilla.org/
todo...

## Raspberry Pi Backups
A guide on how to back up your important RasPi data via ssh to another PC

Since the Backup is based on self written scripts, they can all be found at [this repository](https://github.com/frederikb96/Linux-Scripts)
- Check the repo for up2date information
- The following snippets are only examples
- However this is useful to understand the setup process

### Unison
Use unison as file sync program [unison](https://www.cis.upenn.edu/~bcpierce/unison/)
[Manual for unison](https://www.cis.upenn.edu/~bcpierce/unison/download/releases/stable/unison-manual.html)

Install it via:
```
sudo apt install unison
```

But same version on host and client needed (RasPi), so maybe install .deb manually and download somewhere
https://pkgs.org/download/unison

And unison config for nextcloud backup:
```
# Unison preferences
label = Nextcloud Backup
root = ssh://root@pi4.lan//mnt/data/nextcloud
force = ssh://root@pi4.lan//mnt/data/nextcloud
root = /home/freddy/Backup/nextcloudbackup/data
batch = true
group = false
owner = false
perms = -1
times = true
links = true
retry = 1
logfile = /home/freddy/Computer/Pi_Backup_Log/nextcloud_unison.log
```

And file for backups to external drive
```
# Unison preferences
label = External Backup
root = /home/freddy/Backup
force = /home/freddy/Backup
root = /media/freddy/Backup_Freddy/pi_Backups
ignore = Path lost+found
ignore = Path .Trash-1000
auto = true
group = false
owner = false
perms = -1
times = true
links = true
retry = 1
log = false
```

You can execute unison backup manually via:
```
unison my-profile-name
```

### Backup Scripts
Backup automatically by use of this script
- Just tar all docker folders directly to backup location via pipe
- And use unison to backup large folders
    - no user and group is set to be able to backup to user backup folder on pc
    - so if recovering you have to change the user and group owner after restore on the RasPi, all else is preserved
- The script will write a log to the dedicated folder and if a specific criteria is met, like a long execution time or another error, it will copy the log to the desktop, such that the user of the pc knows that the automatic backup (normally in background) failed

```
#!/bin/bash

#--------------------
# Variables
#--------------------
day=`date +%d`
month=`date +%m`
year=`date +%Y`
hour=`date +%H`
min=`date +%M`

SECONDS=0
maxExecutionTime=5
statusExit=0

adressBackup="/home/freddy/Computer/Pi_Backup_Log/"
adressError="/home/freddy/Desktop/"


#--------------------
# Main
#--------------------

{
echo -e "\n--------------------"
echo "Pi3 Docker:"
echo -e "--------------------\n"
ssh root@pi3.lan 'tar cvf - /opt/docker' > /home/freddy/Backup/pi3/docker.tar
if [ $? -ne 0 ]; then statusExit=1; fi
echo "done"

echo -e "\n--------------------"
echo "Pi4 Docker:"
echo -e "--------------------\n"
ssh root@pi4.lan 'tar cvf - /opt/docker' > /home/freddy/Backup/pi4/docker.tar
if [ $? -ne 0 ]; then statusExit=1; fi
echo "done"

echo -e "\n--------------------"
echo "Pi4 Nextcloud:"
echo -e "--------------------\n"
unison nextcloud_backup > /dev/null
if [ $? -ne 0 ]; then statusExit=1; fi
echo "done"
} >> ${adressBackup}tmp.log 2>&1


#--------------------
# Log Processing
#--------------------

ElM=$((SECONDS / 60))

name="backup_${year}_${month}_${day}_${hour}h_${min}m_ElM$ElM"

find ${adressBackup}* -mtime +30 -type f -name "backup_*" -exec rm {} \;

mv -f ${adressBackup}tmp.log ${adressBackup}$name.log

if [ $statusExit -ne 0 ]
then
    mv -f ${adressBackup}$name.log ${adressBackup}${name}_ERROR.log
    cp ${adressBackup}${name}_ERROR.log ${adressError}
elif [ $ElM -gt $maxExecutionTime ]; then
    mv -f ${adressBackup}$name.log ${adressBackup}${name}_TIME.log
    cp ${adressBackup}${name}_TIME.log ${adressError}
fi

exit $statusExit
```

- save this script somewhere and automatically execute it with a anacron job
	- also see [Linux System (Ubuntu)#Anacron as user]({{< ref "Linux System (Ubuntu)#anacron-as-user" >}})

```
# .anacron/etc/anacrontab: configuration file for anacron user

# See anacron(8) and anacrontab(5) for details.

SHELL=/bin/sh
PATH=/home/freddy/.local/bin/my-bin:/home/freddy/.local/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
HOME=/home/freddy
LOGNAME=freddy

# period  delay  job-identifier  command
1       2       backup-pis.daily                do-backup-pis.sh
```