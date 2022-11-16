---
title: "Linux Backup"
---

# Linux Backup

## Clonezilla Backups
You can use clonezilla to make full backups of your complete drive or the Raspberry SD card
https://clonezilla.org/

## Kopia Backups
- To backup all kind of data I use [Kopia](https://kopia.io)
	- It supports incremental backups, which allow to restore to different points in time
- [Install Kopia](https://kopia.io/docs/installation/#linux-installation-using-apt-debian-ubuntu)
- [Guide on how to use Kopia](https://kopia.io/docs/getting-started/)

- There are basically two backup mechanisms: push and pull
	- With push, the client containing the data is pushing the data to a backup server
	- With pull, the backup server is asking a client to get its data to backup it
- Kopia only supports the push mode (so the client side mode)

- On our client, containing the data, we initialize a repository, which will be located on our backup driver or also backup server
- For a backup drive simply create a repo via
	- `kopia repository create filesystem --path /path/to/backup/drive/folder`
- We create and connect to a backup server repository e.g. via ssh (sftp)
	- We already have to have a working ssh connection from our client to our server with ssh keys set up
	- `kopia repository create sftp --path=/home/freddy/Backup/pi4 --host=freddy-desktop.lan --username=freddy --keyfile=/root/.ssh/id_rsa --known-hosts=/root/.ssh/known_hosts`
- That is all, now we are connected via Kopia to this backup repository
	- Kopia will always reconnect to this reposiotory, except if you manually disconnect at some point via `kopia repository disconnect`
	- Then you can reconnect via `kopia repository connect filesystem --path /path/to/backup/drive/folder`
		- or via `kopia repository connect sftp --path=/home/freddy/Backup/pi4 --host=freddy-desktop.lan --username=freddy --keyfile=/root/.ssh/id_rsa --known-hosts=/root/.ssh/known_hosts`

**Backups**
- Now we can create backups (snapshots) of various data located on our client
- Examples
	- `kopia snapshot create /opt/docker`
	- or if you want to first dump a database from a docker container, to be sure that it is in a valid state when restoring do e.g.:
		- `docker exec postgres-synapse pg_dumpall -U synapse > /databases/docker-synapse-postgres.dump && kopia snapshot create /databases/docker-synapse-postgres.dump && rm /databases/docker-synapse-postgres.dump`
	- `kopia snapshot create /mnt/data/nextcloud`

**Sync Backup to another Backup Repository**
- If we want to sync our backup from one backup repository to another this is also possible with kopia
	- [Forum question about this](https://kopia.discourse.group/t/duplicate-repo-to-another-location/1144/3)
- Connect to the source repository if not already connected
	- `kopia repository connect filesystem --path=/home/freddy/Backup/pi4`
- Now you can sync the repo to the target repository (must be empty folder or have the same `kopia.repository` file) and specify it via path
	- `kopia repository sync-to filesystem --path /media/freddy/Backup_Freddy/backup-server`
	- Here you can also use sftp instead of the filesystem again

**Mount / Restore Backup**
- You can mount the backup to check the files
- `kopia mount snapshot-id ~/my-mount-location`
- `kopia restore snapshot-id ~/my-mount-location`

**Personal Backup Structure**
- On my main server, I have an additional backup drive
- My main server and my other Raspberry Pi both connect to this same backup drive via kopia and do regular backups via cron every 1h
- Since it is risky to only have one backup repository, I also sync the repository to my PC ones per Day via ssh, such that the complete repository is also available at a drive of my PC
- And again I have another external drive, which I sometimes attach to my PC to also sync the repository on my PC to the external drive, to have another 3rd backup location

## Unison Backups

**I recently switched to use Kopia for backups**

Use unison as file sync program [unison](https://www.cis.upenn.edu/~bcpierce/unison/)
[Manual for unison](https://www.cis.upenn.edu/~bcpierce/unison/download/releases/stable/unison-manual.html)

Install it via:
```
sudo apt install unison
```

But same version on host and client needed, so maybe install .deb manually and download somewhere if both Linux versions do not have the same version
https://pkgs.org/download/unison

A unison config for a nextcloud backup:
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

And file for backups to an external drive
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
