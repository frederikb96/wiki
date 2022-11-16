---
title: "Linux System (Ubuntu)"
---

# Linux System (Ubuntu)

## Disks
### Disk info and usage
Display infos:
```
sudo lsblk  # list devices (blocks), simple overview
sudo blkid  # list ids/attributes of devices (blocks)
sudo fdisk -l  # advanced infos and manipulation on disks
```

```
df -h /dev/sda2  # storage usage via command line
```

### Create partitions
WARNING: Partiton table GPT is different than MBR

Adjust partition table with fdisk:
```
sudo fdisk /dev/sda
m for help
p for print current
d delete one
n add new
l list all known types	[1:EFI 10:MSReserved 11:MS basic data 20:Linux]
t change part type
x for extra functions
m for help
n change name [EFI System Partition; Microsoft reserved partition; Basic data partition (Windows)]
f fix order
r return to main menu
q for quit without saving
w for save and quit
```
- use the help menu of fdisk to perform your operations
- after save and quit, a new partition table is created
- But no partitions have been created yet, see next step to do so

Format partitions:
```
sudo mkfs -t ext4 /dev/sd... -L MyName		# Create an ext4 partiton with a label
sudo mkfs.vfat -F32 /dev/sd...				# Create an fvat partiton
sudo mkfs.ntfs /dev/sd...					# Create an ntfs partiton
```

### EFI partition
#### Overview of EFI partition
```
/boot/efi #is the disk
# On disk:
EFI/BOOT/BOOTX64.EFI
EFI/ubuntu/shimx64.efi
EFI/Microsoft/Boot/bootmgfw.efi
```

#### How to restore EFI Partition
Use this if you want to transfer your installed OS to another hard drive and you need to install grub again to boot your OS
- prerequisite is to have already created a new efi partition

Boot into Ubuntu live demo and then:
```
sudo mkdir /mnt/my
sudo mount /dev/sdXY /mnt /my	#mount the root part
sudo mount /dev/sdXX /mnt/my/boot/efi 	#mount efi partition
sudo mount --bind /dev /mnt/my/dev
sudo mount --bind /proc /mnt/my/proc
sudo mount --bind /sys /mnt/my/sys
sudo chroot /mnt/my
grub-install /dev/sdX
grub-update

CTRL-D
sudo umount /mnt/my/dev
sudo umount /mnt/my/proc
sudo umount /mnt/my/sys
sudo umount /mnt/my

# adjust fstab to suit partitions
blkid
sudo nano /mnt/etc/fstab
```

/ was on /dev/sda2 during installation

```
UUID=642af999-3b41-4c99-98fd-a3db5b95e585 /               ext4    errors=remount-ro 0       1
```

/boot/efi was on /dev/sda1 during installation

```
UUID=F492-3210  /boot/efi       vfat    umask=0077      0       1
/swapfile                                 none            swap    sw              0       0
```

more disks maybe ext4 and ntfs are listed:

```
/dev/disk/by-uuid/f44f696e-f96e-41c4-891d-0a61aa83ca77 /mnt/Backup auto nosuid,nodev,nofail 0 0
/dev/disk/by-uuid/0793EAFD5759B133 /mnt/Data auto nosuid,nodev,nofail,uid=freddy,gid=freddy,umask=000 0 0
UUID=e31add1c-c5af-45e8-9b1c-532f7bacf6a3       /home/freddy    auto    defaults        0       2
UUID=f44f696e-f96e-41c4-891d-0a61aa83ca77       /home/freddy/Backup     auto    defaults,nofail 0       2
```

#### bootmgr for EFI
Some useful functions to manipulate efi bootmgr 
```
sudo efibootmgr -c -d /dev/sda -p 1 -l \\EFI\\ubuntu\\grubx64.efi -L UbuntuMain -b 0000
sudo efibootmgr -c -d /dev/sdb -p 1 -l \\EFI\\Microsoft\\Boot\\bootmgfw.efi -L WindowsMain -b 0002
sudo efibootmgr -b xxxx -a
sudo efibootmgr -o 0000,0002,0001
```

### Changing drives and fixing UUID
If you have your installation on one drive and want to transfere it e.g. via clonezilla to another external drive, check that in fstab the correct deivce uuids are set, if not done by clonezilla automatically

```
sudo blkid
```

get the PARTUUID, like boot: ae2bd6d7-01 and root ae2bd6d7-02

then adjust in
```
sudo nano /etc/fstab
```

example for Raspberry:
```
proc            /proc           proc    defaults          0       0
PARTUUID=ae2bd6d7-01  /boot           vfat    defaults          0       2
PARTUUID=ae2bd6d7-02  /               ext4    defaults,noatime  0       1
```


### LVM and LUKS encrypted

#### Encrypt external drives
A guide for external devices https://opensource.com/article/21/3/encryption-luks
- Option 1: You can also use Ubuntu disk manager to create external encrypted drives
	- You can also use it to mount devices
- Option 2: Command line version:

```
sudo cryptsetup luksFormat /dev/sdX
sudo cryptsetup open /dev/sdX mydrive
# sudo cryptsetup close mydrive
sudo mount /dev/mapper/mydrive /mnt/mydrive
```

#### Encrypt Ubuntu
- During installation of ubuntu, choose to erase the whole disk and during this step press the advanced button
- then check the two boxes to use lvm as volume manager and choose to encrypt the disk

#### Overview LVM and Luks
[guide on lvm](https://linuxhandbook.com/lvm-guide/)

| Disk Partitioning System | LVM             |
| ------------------------ | --------------- |
| Partitions               | Logical Volumes |
| Disks                    | Volume Groups   |
- Physical volumes are not so easy to represent in this comparison

LVM components:
1. Physical Volumes = pv
2. Volume Groups = vg
3. Logical Volumes = lv

Example overview of Ubuntu if using luks encrypted partition and lvm
```
nvme0n1               259:0    0 119,2G  0 disk  
├─nvme0n1p1           259:1    0   512M  0 part  /boot/efi
├─nvme0n1p2           259:2    0   1,7G  0 part  /boot
└─nvme0n1p3           259:3    0 117,1G  0 part  
  └─nvme0n1p3_crypt   253:0    0 117,1G  0 crypt 
    ├─vgubuntu-root   253:1    0 116,1G  0 lvm   /
    └─vgubuntu-swap_1 253:2    0   980M  0 lvm   [SWAP]
```

- We have the real partition on the drive nvme0n1p3
    - we cannot access it like this, because it is encrypted
    - so we have to decrypt it and map it somewhere
	    - `sudo cryptsetup open /dev/nvme0n1p3 nvme0n1p3_crypt`
	    - now we have the decrypted part sitting on top of the real partition
	    - in real it is mapped to /dev/mapper/nvme0n1p3_crypt
- Further the mapped crypt parition represents also our lvm physical volume
    - `sudo pvs`
- On top of this physical volume their sits the volume group `vgubuntu`
    - `sudo vgs`
- And the group itself contains two lvm logical volumes, the root volume and the swap partition
    - `sudo lvs`
- The logical volumes itself contain the real file system, which can be of any type
    - Note: the underlying file system must not be the same size as the logical volume where it is placed in, so take care, that they are always the same size, such that no erros come up


Commands:
```
sudo pvs
sudo vgs
...
```

#### Resize logical volumes
[guide on shrinking volumes](https://www.seimaxim.com/linux/how-to-shrink-an-lvm-logical-volume)

shirnk:
- unmount the logical volume if it is mounted
    - `sudo umount /dev/vgubuntu/root`
- resize
    - Resize the underlying filesystem with -r flag and also automatically the logical volume (-L for the desired size)
    - `sudo lvreduce -r -L 80G /dev/vgubuntu/root`

### Swap
Section about Swap files and partitions

#### Swapfile
See [here](https://linuxhandbook.com/increase-swap-ubuntu/) or here [how to create new](https://linuxize.com/post/create-a-linux-swap-file/)

check current state
```
swapon --show
free -h
```

change old
```
sudo swapoff /swapfile
```

create or increase
```
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
```

enable it
```
sudo swapon /swapfile
```

Check if new created that it gets mounted
```
sudo nano /etc/fstab
```
Must look somehow like this
```
/swapfile swap swap defaults 0 0
# or like this for fresh Ubuntu e.g.
/swapfile                                 none            swap    sw              0       0
```

#### Swap partition
Swap partition is registerd in fstab
```
sudo nano /etc/fstab
```

```
/dev/mapper/vgubuntu-swap_1 none            swap    sw              0       0
```

##### LUKS
You can use the above information about [#Resize logical volumes]({{< ref "#resize-logical-volumes" >}}) to change the size of your swap partition if LUKS encrypted

- Boot into a live usb stick or something similar where your desired disk does not need to be mounted
- `sudo cryptsetup open /dev/sda3 mydrive`
- check that everything is correct and you use the right names
    - `sudo lsblk`
    - `sudo lvs`
- `sudo lvresize -r -L -16G /dev/vgubuntu/root `
- automatic resize is for swap partition not possible
    - `sudo lvresize -l +16G /dev/vgubuntu/swap_1`
    - so manually format again
    - `sudo mkswap /dev/vgubuntu/swap_1 `
- done reboot


### Prevent file system checks on boot
Not working, sadly
```
sudo tune2fs -c 0 /dev/sda2
```


## Setup

### Transmit home to new installation
You can backup your old home directory, setup your new OS and transmit it back again
- Backup old home
	- Mount your external drive and cd into
		- `cd /media/mybackupdrive`
		- `rsync -avHP /home/freddy ./`
		- folder freddy will be backuped to drive
- Install your new OS
- Log into your new OS (your user must have the same uid as the one from the backup)
	- this is normally the uid 1000 for the first user
	- `id`
- Create a new user in the OS (e.g. name it "tmp")
- Log out and log in to the new user
- Now you can mv your home directory of the real user away
	- `mv /home/freddy /home/freddy.bak`
- And mount your external drive and get all back
	- `cd /media/mybackupdrive`
	- `rsync -avHP ./freddy /home/`
- Now log into the other user again
- And delete the tmp user including its home directory


### Ubuntu Server
- I encrypt my ubuntu server va LUKS
- [Guide on how to install ubuntu server and setup remote login with luks enabled](https://blog.gradiian.io/migrating-to-cockpit-part-i/)
	- Or this [very simple remote login lukes guide](https://blog.gradiian.io/migrating-to-cockpit-part-i/)
- Basically I install `sudo apt install dropbear-initramfs
- Copy your public key from your PC to the dropbear folder
	- `sudo cp ~/.ssh/authorized_keys /etc/dropbear/initram/`
- Configure dropbear
	- `sudo nano /etc/dropbear/initram/dropbear.conf`
	- `DROPBEAR_OPTIONS="-s -p 21 -c cryptroot-unlock"`
	- do not accept passwords, use port 21 and automatically start decryption
- Update `sudo update-initramfs -u`

If you have another disk, setup that it gets automatically decrypted once you decrypted the main disk
- We can sue [crypttab](https://linuxconfig.org/introduction-to-crypttab-with-examples) for this
- We need a [keyfile](https://access.redhat.com/solutions/230993), which can be used to decrypt the disk
	- `dd if=/dev/random bs=32 count=1 of=/sda.keyfile`
	- `cryptsetup luksAddKey /dev/sda /sda.keyfile`
- Now specify this in `sudo nano /etc/crypttab`
	- `dm_crypt-1 UUID=82ff1267-2d02-4147-aad8-261e1abe3c08 /sda.keyfile luks`

Now you are ready to reboot and log into the server to decrypt it:
- `ssh -p 21 root@ubuntu-server`
- Enter the password
- Wait and connect normally to server
- `ssh root@ubuntu-server`

### Linux-Surface
Follow this guide [surface installation](https://github.com/linux-surface/linux-surface/wiki/Installation-and-Setup), basically for SP4:
- install ubuntu
- install linux-surface:
```
wget -qO - https://raw.githubusercontent.com/linux-surface/linux-surface/master/pkg/keys/surface.asc | gpg --dearmor | sudo dd of=/etc/apt/trusted.gpg.d/linux-surface.gpg

echo "deb [arch=amd64] https://pkg.surfacelinux.com/debian release main" | sudo tee /etc/apt/sources.list.d/linux-surface.list

sudo apt update
sudo apt install linux-image-surface linux-headers-surface iptsd libwacom-surface
sudo systemctl enable iptsd

sudo apt install linux-surface-secureboot-mok

sudo update-grub

# Post installation
sudo apt install intel-microcode linux-firmware

# Reboot and check kernel
uname -a
```

**Problem I encountered when installing linux-image-surface**
```
# Error
Setting up linux-image-5.19.2-surface (5.19.2-surface-1) ...
update-initramfs: Generating /boot/initrd.img-5.19.2-surface
W: Possible missing firmware /lib/firmware/i915/skl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/bxt_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/kbl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/glk_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/kbl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/kbl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/cml_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/icl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/ehl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/ehl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/tgl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/tgl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/dg1_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/tgl_guc_70.1.1.bin for module i915
W: Possible missing firmware /lib/firmware/i915/adlp_guc_70.1.1.bin for module i915
I: The initramfs will attempt to resume from /dev/dm-2
I: (/dev/mapper/vgubuntu-swap_1)
I: Set the RESUME variable to override this.
```

After installation you can check error again with:
```
sudo update-initramfs -k all -u
```

Fix it with:
- Need linux firmware from internet, since newer then the one form linux-surface
- https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/
- and copy the downloaded files over
```
sudo cp /home/freddy/Downloads/linux-firmware-20220815/i915/*70* /lib/firmware/i915/
```
files are:
```
-rw-r--r-- 1 root root 289472 Aug 25 21:27 adlp_guc_70.1.1.bin
-rw-r--r-- 1 root root 206464 Aug 25 21:27 bxt_guc_70.1.1.bin
-rw-r--r-- 1 root root 206976 Aug 25 21:27 cml_guc_70.1.1.bin
-rw-r--r-- 1 root root 265152 Aug 25 21:27 dg1_guc_70.1.1.bin
-rw-r--r-- 1 root root 365568 Aug 25 21:27 dg2_guc_70.1.2.bin
-rw-r--r-- 1 root root 369600 Aug 25 21:27 dg2_guc_70.4.1.bin
-rw-r--r-- 1 root root 274496 Aug 25 21:27 ehl_guc_70.1.1.bin
-rw-r--r-- 1 root root 206784 Aug 25 21:27 glk_guc_70.1.1.bin
-rw-r--r-- 1 root root 274496 Aug 25 21:27 icl_guc_70.1.1.bin
-rw-r--r-- 1 root root 206976 Aug 25 21:27 kbl_guc_70.1.1.bin
-rw-r--r-- 1 root root 206208 Aug 25 21:27 skl_guc_70.1.1.bin
-rw-r--r-- 1 root root 277440 Aug 25 21:27 tgl_guc_70.1.1.bin
```

#### Surface Firmware
- https://www.reddit.com/r/Surface/comments/no3rz4/how_to_fix_surfacepro4_type_cover_issue_battery/
- After over install was necessary to repeat procedure with delete driver and reboot and than call same launcher again to be able to uninstall. Then install desired.
- Links to this website on how to do it with Linux somehow: https://github.com/linux-surface/surface-uefi-firmware and https://github.com/linux-surface/surface-uefi-firmware/issues/12 on how to fix maybe if not working

#### Surface IPTS Touch
```
sudo gedit /etc/ipts.conf
```
- 16 looks good

Commands for iptsd
```
sudo systemctl stop iptsd.service
sudo systemctl start iptsd.service
sudo systemctl restart iptsd.service;
sudo iptsd-reset-sensor
```
If touch doesn't work, try this:
```
sudo killall iptsd
sudo rmmod ipts
sleep 5
sudo modprobe ipts

```

## Basic Installs

```
sudo apt update && sudo apt upgrade -y && sudo apt autoremove

sudo apt install -y build-essential git python3 python3-pip curl adb fastboot cmake

sudo apt install -y nautilus-admin synaptic gnome-screenshot gnome-shell-extension-manager gnome-tweaks menulibre meld

sudo apt install -y evolution evolution-ews evolution-plugins
sudo apt autoremove -y thunderbird*

# Fix
sudo apt install -y libcanberra-gtk-module

# Other
sudo apt -y install steam
sudo apt install texlive-latex-extra texlive-lang-german texlive-lang-english
sudo apt install texmaker
```

### Extensions
Some useful gnome extensions:
- AATWS
- Screenshot Tool
- Sound Output Chooser

## Programs
Some Programs I do not simply install via apt and some other related information

### Launchers
Use launchers to start applications, which do not automatically create a launcher and a Application Menu entry.

- You can place user launchers here: `~/.local/share/applications/`
- It is easy to create launchers with the menulibre application `sudo apt install menulibre`
- More details available under the [specification specs](https://specifications.freedesktop.org/desktop-entry-spec/desktop-entry-spec-latest.html)
- Force an update of launchers by `update-desktop-database ~/.local/share/applications/`

### Snap
Some general snap commands I often forget
```
sudo snap install mysnap
snap connections --all
snap connections mysnap
snap connect <mysnap>:<plug interface> :<slot interface>
```

Backups under:
```
/var/lib/snapd/snapshots
```

Examples:
```
sudo snap install clion --classic
sudo snap install pycharm-professional --classic
sudo snap install gitkraken --classic
sudo snap install xournalpp
sudo snap install discord
sudo snap install element-desktop
```

### Flatpak
Install:
```
sudo apt install flatpak
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install my.flat.pak
flatpak run my.flat.pak
```

You can create anacron job to automatically update flatpak (not part of ubuntu auto updates yet)
- also see [#Anacron as user]({{< ref "#anacron-as-user" >}})
```
5	5	flatpak_update.weekly	(gnome-terminal -- bash -c 'flatpak update')
```

Examples:
```
flatpak install us.zoom.Zoom
```

### Kdocker / App Tray
You can use kdocker (`sudo apt install kdocker`) to force applications to the tray. This is useful for mail programs like evolution, which do not support tray by default.

Example for app launcher with automatic tray function for xorg:
```
[Desktop Entry]
Version=1.1
Type=Application
Name=Mail Tray
Comment=Starts mail and sends it to tray.
Icon=evolution-symbolic
Exec=bash -c "if [ $( pgrep -x evolution ) ] ; then evolution; else kdocker -q -- evolution -c mail; fi"
Actions=
Categories=Email;Utility;
```

Kdocker is broken for Wayland, but by launching the apps as xorg ones, it is still possible with this workaround

Launcher:
```
[Desktop Entry]
Version=1.1
Type=Application
Name=Mail Tray
Comment=Starts mail and sends it to tray for wayland.
Icon=evolution-symbolic
Exec=bash -c "if [ $( pgrep -x evolution ) ] ; then GDK_BACKEND=x11 evolution; else my-mail-tray-wayland.sh; fi"
Actions=
Categories=Email;Utility;
```
And `my-mail-tray-wayland.sh` script, which must be on your path:
```
#!/bin/bash

GDK_BACKEND=x11 evolution -c mail &
sleep 1
pid=$( pgrep -x evolution )
QT_QPA_PLATFORM=xcb kdocker -q -x $pid
```



### Obsidian
[Download Obsidian AppImage](https://obsidian.md/)
- Place it in `~/Programs/Obsidian.AppImage`
- Create desktop launcher which also enables obsidian URI
```
[Desktop Entry]
Version=1.1
Type=Application
Name=Obsidian
Icon=obsidian
Exec=bash -c "/home/$(id -un 1000)/Programs/Obsidian.AppImage %u"
Terminal=false
StartupWMClass=obsidian
MimeType=text/html;x-scheme-handler/obsidian;
```
- .
	- Update launchers `update-desktop-database ~/.local/share/applications/`
- Now you can use URI calls like
	- `xdg-open "obsidian://open?path=%2Fhome%2Ffreddy%2FNextcloud%2FNotes%2FSachen"`
	- Only working if vault is already opened once manually, such that it is registerd as a valid vault
- Now we want to enable opening of md files via Obsidian as default program
	- Create a new script, which will link files, which are not in vaults to a tmp vault, such that they can be opened
		- For this first manually create a ObsidianTmp vault
		- and get the my-obsidian.py script [from my linux-scripts repo](https://github.com/frederikb96/Linux-Scripts)
		- change your path to your ObsidianTmp vault
		- this script will call the previous created obsidian launcher via the URI method
	- Now create a new launcher, which will call the new script and pass the filename
```
[Desktop Entry]
Version=1.1
Type=Application
Name=Obsidian Auto
Icon=obsidian
Exec=my-obsidian.py %u
Terminal=false
```
- .
	- Now you can right click on of your md files and change the default program to your new Obsidian Auto launcher

### Signal
- https://signal.org/download/
- Use the flag to start signal in tray `--use-tray-icon`

### Typora
```
wget -qO - https://typora.io/linux/public-key.asc | sudo tee /etc/apt/trusted.gpg.d/typora.asc
sudo add-apt-repository 'deb https://typora.io/linux ./'
sudo apt update
sudo apt install typora pandoc
```

### Nextcloud
```
sudo add-apt-repository ppa:nextcloud-devs/client
sudo apt update
sudo apt install -y nextcloud-client nautilus-nextcloud
```

### OnlyOffice
https://www.onlyoffice.com/de/download-desktop.aspx
- Download .deb

### Masterpdf
https://code-industry.net/masterpdfeditor/
- Download .deb and install

### Anaconda
- Download on homepage, chmod +x and execute script as user
- Install to .anaconda3, and start ini
- adds anaconda to path in `.bashrc`
- Conda base enviroment gets activated on startup, disable by `conda config --set auto_activate_base false`
- Activate conda enviroment with `conda activate base`
    - `conda create --name myenv` creates new empty env, you can choose .anaconda3/envs as folder
    - conda list to show packages
    - pip install if conda activated to install packages not available otherwise
    - conda bin is located under ???

### Matlab
Download and install as user into programs (do not install as root, only has disadvantages)
- link scripts to .local/bin during installation (do not link to the root bin)
- create app launcher and maybe use my matlab.py script, which allows opening of files via explorer
	- [from my linux-scripts repo ](https://github.com/frederikb96/Linux-Scripts)
- logout and log in again
- optionally import old settings to .matlab
- start matlab

### Webcatalog
https://webcatalog.io/
- Download AppImage and place in some folder
- create a launcher for it
	- you can also use the one [from my linux-scripts repo ](https://github.com/frederikb96/Linux-Scripts)

### Webapp
**Not working with new snap firefox**; use the [#Webcatalog]({{< ref "#webcatalog" >}}) instead

https://www.michlfranken.de/ubuntu-22-04-mit-firefox-esr-problem-mit-webapps-manager-mit-loesungsweg/
```
sudo dpkg -i webapp-manager_1.1.9_all.deb 
```
```
sudo apt --fix-broken install
```

### Virtual Box
https://www.virtualbox.org/wiki/Downloads
```
# Use manual download method via .deb download and install it
# Also downlaod extensions pack there and install
```

- You can connect the webcam simply via the devices interface; needs connection every time you start the VM
- Or use following commands to do it from the terminal:
```
VBoxManage list webcams
VBoxManage controlvm Windows10 webcam attach .1
VBoxManage controlvm Windows10 webcam list
```


### VMware Workstation Player
I would recommend VirtualBox, much better lately for Linux
https://www.vmware.com/products/workstation-player.html
https://docs.vmware.com/en/VMware-Workstation-Player-for-Linux/16.0/com.vmware.player.linux.using.doc/GUID-BF62D91D-0647-45EA-9448-83D14DC28A1C.html

Fix error with vmmon
https://kb.vmware.com/s/article/2146460
Tried also this:
- `sudo apt install open-vm-tools-desktop open-vm-tools reboot`
- `sudo apt install libaio1`


### Tikzit
- https://tikzit.github.io/
- Move tikzit folder to programs
- change the path to your tikzit bin within the share/application subfolder
- execute the install-local.sh script
- done...

### Tailscale VPN
See under [Raspberry Pi#Tailscale VPN]({{< ref "Raspberry Pi#tailscale-vpn" >}}) on how to connect to your local network via tailscale vpn

### Work Programs
Some Programs I need at work

#### VPN Openconnect Cisco
Option1: use rwth one form software shop https://help.itc.rwth-aachen.de/service/vbf6fx0gom76/

Option2: use openconnect to connect to cisco vpn integrated into linux
- Install the openconnect network manager
    - for gnome use this one
    - `sudo apt install network-manager-openconnect-gnome`
- Start advanced network manager via launcher or via terminal `nm-connection-editor`
- Create a new openconnect vpn connection
- choose a name and set the gateway `vpn.rwth-aachen.de`, do not set anything else and click save
- start the vpn via your network settings
- you will be asked for your username and the password
    - you can also check the box to store the password

#### Sciebo
```
wget -nv https://www.sciebo.de/install/linux/Ubuntu_22.04/Release.key -O - | sudo tee /etc/apt/trusted.gpg.d/sciebo.asc
echo 'deb https://www.sciebo.de/install/linux/Ubuntu_22.04/ /' | sudo tee -a /etc/apt/sources.list.d/owncloud.list
sudo apt update
sudo apt install -y sciebo-client sciebo-client-nautilus
```

#### Zotero
[website zotero](https://www.zotero.org/support/installation)
[zotero deb support](https://github.com/retorquere/zotero-deb)

```
wget -qO- https://raw.githubusercontent.com/retorquere/zotero-deb/master/install.sh | sudo bash
```

```
sudo apt update
```

```
sudo apt install zotero
```

**Uninstall**

```
wget -qO- https://raw.githubusercontent.com/retorquere/zotero-deb/master/uninstall.sh | sudo bash
```

```
sudo apt purge zotero
```

**Setup**

We have two systems:
- Data folder containing local library
    - it also contains the pdfs
- Zotero cloud
    - containing library without media via their cloud
    - and link to media folder in my cloud, which only contains the pdfs

Setup:
- Start Zotero and choose in settings as data folder, the one I synced via cloud
    - delete the one in home directory
    - syncing the local data folder is only additional, since media folder is synced also, which contains pdfs
- Start sync by connecting to your zotero account
    - Also enable sync my own media via webdav and enter the address to cloud resources zotero folder
- done...

#### Wireshark
```
sudo apt install wireshark tshark
sudo dpkg-reconfigure wireshark-common 
sudo usermod -a -G wireshark $USER
# Log out to be ok
gnome-session-quit --logout --no-prompt
```

#### Debugging tools
```
sudo apt install -y openocd gcc-arm-none-eabi libncurses5
```

#### Boxycryptor
- Use portable linux version
- Move to Programs and create launcher via menulibre

#### ChibiStudio
- Also manually copy folder to home/user
- create launcher

#### ProM
- Move the ProM folder which contains the .sh script to your Programs directory
	- e.g. `~/Programs/Prom`
- it stores data in
	- `~/.ProM611`
	- `~/.UITopia`
- you need Java to get it working
	- `sudo apt install openjdk-8-jdk`
- create a launcher
- You need to manipulate the .sh script with your Java path
	- open it 
	- `gedit ~/Programs/Prom/ProM611.sh`
	- and add the following lines at top
```
cd $(dirname $0)
JAVA=/usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java
```

#### RapidMiner
- Move extracted folder somewhere like `~/Programs/RapidMiner`
- it stores data in
	- `~/.RapidMiner`
- Java 8 is necessary so install it parallel to your current java by:
- `sudo apt install openjdk-8-jdk`
- `java --version`
- Should still display previous one and now change environment variable in
- `gedit ~/Programs/RapidMiner/RapidMiner-Studio.sh`
- by adding this line in beginning (check that version is correct)
- `JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64`
- change also 3 lines accoriding to https://community.rapidminer.com/discussion/comment/67912/#Comment_67912
- create an application launcher

#### QT
Option 1:
- Download installer https://www.qt.io/download
- Choose the open-source version
- Make installer executable and run as user and select your installation folder e.g. `/home/user/Programs/QT`
- Select custom installation as installation method
- Select components, check default
	- additionally add a Qt installation like 6.4.0
- Install

Option 2:
- Use QT and QT Creator provided by your package manager
- `sudo apt install qtcreator cmake qmake6 qt6-base-dev qml-qt6 libqt6charts6-dev`
- Optionally for Wayland: `sudo apt install qt6-wayland-dev`

Post setup:
 - Ubuntu
	- Install also `sudo apt install mesa-common-dev` for graphic compatibility
- Now you can launch QT Creator
- Under settings, kits, the desktop kit should be available and configured with your QT installation
	- If that is not the case manually add your qmake executable under the QT Versions tab
	- Linux: located under `/usr/bin/qmake6`

##### CLion Alternative
- You can also use CLion to compile and debug your Qt project
- Check the [official guide from CLion](https://www.jetbrains.com/help/clion/qt-tutorial.html)
- You basically have to import the project as a CMake project
	- the `CMakeLists.txt` must be created for this
- Go to CLion settings and under Build and CMake add the Qt prefix path to your CMake options
	- `-DCMAKE_PREFIX_PATH="~/path-to-your-installation/Qt/6.4.0/gcc_64"`
	- This works best, if you have installed Qt via "Option 1"
- Now you have to add the modules to the `CMakeLists.txt`
```
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)

find_package(Qt6 COMPONENTS Core Widgets Charts REQUIRED)

...

target_link_libraries(your-program Qt6::Core Qt6::Widgets Qt6::Charts)

```

## System

### Upgrade Ubuntu
```
update-manager -c
```

### Gnome Restart
```
sudo systemctl restart gdm3.service
```

### Path, Scripts, profile, bashrc
You can add stuff to your PATH in  `~/.profile`\:
```
# set PATH so it includes my custom private scripts if it exists
if [ -d "$HOME/.local/bin/my-bin" ] ; then
    PATH="$HOME/.local/bin/my-bin:$PATH"
fi
```

Path is also handled in `.bashrc`, just more bash specific stuff in there
in .bashrc for example:
```
# My aliases and functions
function mfind { find -iname \*$1\*; }
alias mgrep='grep -rl -e'
```

### Anacron as user
- [scheduled - How can I run anacron in user mode? - Ask Ubuntu](https://askubuntu.com/questions/235089/how-can-i-run-anacron-in-user-mode)
- [Running anacron as a user – Come and Tech it !](https://comeandtechit.wordpress.com/2019/06/17/running-anacron-as-a-user/)
- enable anacron in `~/.profile`
```
# User anacron 
/usr/sbin/anacron -s -t $HOME/.anacron/etc/anacrontab -S $HOME/.anacron/spool
```
- Create `mkdir -p .anacron/{etc,spool}`
- copy default `cp /etc/anacrontab .anacron/etc`
- In `~/.anacron/etc/anacrontab`
```
# .anacron/etc/anacrontab: configuration file for anacron user 

# See anacron(8) and anacrontab(5) for details. 
SHELL=/bin/sh 
PATH=/home/freddy/.local/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
HOME=/home/freddy
LOGNAME=freddy 

# period delay job-identifier command
1 2 my-custom-name.daily my-script.sh
```
- anacron is using its own environment variables, so take care to set those correctly
    - maybe add your own paths to the PATH variable

### systemd user service
- [Creating a Simple Systemd User Service](https://blog.victormendonca.com/2018/05/14/creating-a-simple-systemd-user-service/)
	- Useful if you want to automate some scripts
- Look at `~/.config/systemd/user`
```
[Unit]
Description=[Service description]

[Service]
Type=simple
StandardOutput=journal
ExecStart=[script path]

[Install]
WantedBy=default.target
```

Example for service, would be to have a service, which checks if a specific drive gets mounted, in this case to something, see also [#Detect Drives Attached]({{< ref "#detect-drives-attached" >}})
```
[Unit]
Description=SecondBackup
Requires=media-freddy-SecondBackup.mount
After=media-freddy-SecondBackup.mount

[Service]
ExecStart=/home/freddy/.local/bin/my/doowncloudautosecondbackup.sh

[Install]
WantedBy=media-freddy-SecondBackup.mount
```

### Detect Drives Attached
- [usb - How to run a script when a specific flash-drive is mounted? - Ask Ubuntu](https://askubuntu.com/questions/25071/how-to-run-a-script-when-a-specific-flash-drive-is-mounted)

Quotes:

There's much nicer solution with **systemd** now. You create a service which depends and is wanted by you media e.g.: `/etc/systemd/system/your.service`
```
[Unit]
Description=My flashdrive script trigger
Requires=media-YourMediaLabel.mount
After=media-YourMediaLabel.mount

[Service]
ExecStart=/home/you/bin/triggerScript.sh

[Install]
WantedBy=media-YourMediaLabel.mount
```

Then you have to start/enable the service:
```
sudo systemctl start your.service
sudo systemctl enable your.service
```

After mount, systemd fires your trigger script. The advantage over udev rule is that the script really fires after mount, not after adding system device.

**Use case**\: I have an encrypted partition which I want to backup automatically. After adding the device I have to type in the password. If I hooked the backup script to udev, the script attempts to run at the time when I'm typing password, which will fail.

Resource: [Scripting with udev](http://jasonwryan.com/blog/2014/01/20/udev/)

**Note:** You can find your device unit with:
```
systemctl list-units -t mount
```


### Enable Hibernation
https://ubuntuhandbook.org/index.php/2021/08/enable-hibernate-ubuntu-21-10/

```
df -h
blkid
sudo filefrag -v /swapfile #get first physical address
sudo gedit /etc/default/grub #add resume=UUID=xxx resume_offset=xxx to spalsh...
sudo update-grub
reboot
```
- Download gnome extension to enable the Hibernation button next to reboot and shutdown...

### Root user display x server xhost
Add root to display
```
xhost +SI:localuser:root
```

### apt-key deprecated
Use this instead, replace url and name of e.g. owncloud.asc resulting file
```
wget -qO- https://download.owncloud.com/desktop/ownCloud/stable/latest/linux/Ubuntu_22.04/Release.key | sudo tee /etc/apt/trusted.gpg.d/owncloud.asc
```

### Updates, sources, keyrings
Keyrings can be stored under:
```
/usr/share/keyrings/
# e.g
/usr/share/keyrings/docker-archive-keyring.gpg
```

Sources located here:
```
/etc/apt/sources.list.d/
/etc/apt/sources.list.d/docker.list #example for one app
```

### Canberra GTK Module fix
```
sudo apt-get install libcanberra-gtk-module
```

### Disable BIOS ACPI warnings via grub
[here explaine](https://askubuntu.com/questions/1416198/ubuntu-22-04-acpi-bios-error-bug-could-not-resolve-symbol-errors-on-asus-x7)
```
sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3"
sudo update-grub
```

### Fingerprint authentification
```
sudo pam-auth-update
```

### Disable root need for command
```
sudo visudo
```

add the following to the file
```
%sudo ALL=NOPASSWD:/usr/sbin/pm-suspend
%sudo ALL=NOPASSWD:/usr/sbin/pm-hibernate
```
now you can execute `sudo pm-suspend` without being asked for a password

## Network
For information about network commands and vpn look at the dedicated page about [Network]({{< ref "Network" >}})

## Commands
Useful commands I often need and forget about

### Find without errors
`find / -name "*search*" 2>/dev/null`

### Find Files and Replace Text
`find ./ -name "*.m" -exec sed -i 's/TextFind/TextReplace' {} \;`

### Find Files containing Text - Grep
`grep -rl -e "doowncloudautosecondbackup" /`

### Text Multiline Replace
```
print("Open file")
	file = open("./cal.ics", "r")
	text = file.readlines()
	file.close()

	print("Start process")
	i = 0
	line_start = -1
	found_identifier = False
	matches = ["Datenstrukturen", "Datenbanken", "Datenkommunikation", "Business", "Reinforcement", "Algorithmic"]
	for k in range(len(text)):

		# Check for start phrase
		if text[i].__contains__("BEGIN:VEVENT"):
			line_start = i

		# if start phrase found, check for further cases
		if line_start != -1:
			# check if identifier found
			if any(x in text[i] for x in matches):
				found_identifier = True

			# check if already found end phrase
			if text[i].__contains__("END:VEVENT"):
				# if identifier found, delete the lines
				if found_identifier:
					# delete lines
					text = text[:line_start] + text[i+1:]
					# fix index, since lines deleted
					i -= (i - line_start + 1)
				# Reset line Start again
				line_start = -1
				found_identifier = False
		i += 1

	# Save new text
	file_out = open("cal_new.ics", "w")
	file_out.writelines(text)
	file_out.close()
	print("Saved")
```

### Change time of files
Useful for summer and winter time fix
`find ./ -type f -exec bash -c 'touch -m --date="$(stat -c %y "$1") - 3600 sec" "$1"' -- {} \;`

### ADB sync
adb root
- from phone to pc
	- `adb-sync -dfR sdcard/Music/Media/ /mnt/Backup/owncloud/Music/ --dry-run`
- from pc to phone
	- `adb-sync -df /mnt/Backup/owncloud/Music/ sdcard/Music/Media/ --dry-run`

### Rsync Android SSH
Use Rsync via SSH to Android
1. start [sshelper](https://arachnoid.com/android/SSHelper/)
2. `rsync -vrp --delete --ignore-existing --omit-dir-times --no-perms --inplace -e 'ssh -p 2222' /mnt/Backup/owncloud/Music/ 192.168.0.59:SDCard/Music/Media/`
(meanings: verbose, recursive, progress...)

### Rsnyc SSH Root
`rsync -rthv --rsync-path="sudo rsync" pi@192.168.0.129:source dest`
- some rsync flags I always forget the meaning of
	- a:archive (-rlptgoD), r:recursive, t:time, l:symlinks, p:permission, g:group, o:owner, D:device?, H:hardlinks, h:humanReadable, v:verbose, A:acls, X:extendedAttrb, P:progressAndPartial

### Terminal prevent disappear
```
gnome-terminal -- bash -c "ls & read"
```

### Tar
Compress and extract to a folder
```
tar cf ./some_folder/file_compressed.tar -C ./some_folder file_original
tar xf ./some_folder/file_compressed.tar -C ./some_folder
```

Via network
```
ssh root@pi4.lan 'tar cf - /opt/docker' > /home/freddy/Backup/pi4/docker.tar
```

### Nautilus ssh
```
sftp://root@my.server.com/
# OR, the same:
ssh://root@my.server.com/
```

### Generate SSH Keys
```
ssh-keygen -f ~/.ssh/my-new-key -b 4096
```

### Nautilus Error Searching - Clear Cache
If Nautilus crashes on searching:
`tracker3 reset -rs`