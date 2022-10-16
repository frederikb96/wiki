---
title: "Raspberry Pi"
---

# Raspberry Pi

## General
- Some security advices: [security guide](https://raspberrytips.com/security-tips-raspberry-pi/)

## Basic setup

### Install

Use raspberry imager to install RasPi OS to sd card

Recommend:
- Use 64 bit version, has some compatibility advantages with programs only running on 64 bit
- in settings where you can pre configure settings for the OS:
    - use your public ssh key from your PC `~/.ssh/id_rsa.pub` to disable password authentication for the pi user and use ssh instead
- maybe change other settings as you like (maybe static ip)
	- Though, I like to give the RasPi a static IP via my router interface, instead of forcing the router via the RasPi DHCPcd.conf to give him its desired IP address
- If you have a external drive and want to increase the speed of the RasPi, flash OS to the drive and boot RasPi via this instead of the sdcard
	- Link to guide https://www.makeuseof.com/boot-raspberry-pi-4-via-ssd-network/
    - You can use the imager and just choose your external drive where to flash the OS to, which must be connected to your PC
    - You also have to enable the Pi to boot from a USB port first as described in the guide, by flashing the boot loader program to an SD card and start the Pi once with this SD card
    - Then remove SD card and connect external drive instead and power Pi up
    - SD card speed is always slower than external drives via USB 3.0 (most of the time even via USB 2.0)

Connect to the Pi via ssh
```
ssh pi@your.ip.for.raspi
```

### SSH via local ethernet directly to Pi
https://raspberrypi.stackexchange.com/questions/3867/ssh-to-rpi-without-a-network-connection
- a guide on how to connect to the Pi if you do not have a router or switch available
- I guess it is only working, if RasPi uses dynamic ip, so do not force static ip during installation via dhcpcd.conf for this to work

### Static IP
I use the interface of my router to assign the RasPi a static ip address (see manual for your router)
- I once had problems with ipv6 if I force a static local ipv4 address via the dhcpcd.conf
- In my router settings I use addresses like ...101 and ...102 for my RasPis (just as an example to easily keep track of them)

#### dhcpcd.conf
If you want to use the dhcpcd.conf anyways - has the advantage that you can connect the RasPi to any router and immediately now the ip adress
```
sudo nano /etc/dhcpcd.conf
```

```
# Example static IP configuration:
interface eth0
static ip_address=192.168.0.101/24
#static routers=192.168.0.1
#static domain_name_servers=192.168.0.1
```

### DNS
here a [a guide about DNS for RasPi](https://pimylifeup.com/raspberry-pi-dns-settings/)
You can also check out the general guide for DNS servers: [Network#DNS Global]({{< ref "Network#dns-global" >}})

If you do not want to use the default DNS server of your router (which is normally the one of your internet provider), you can directly assign a DNS server to the RasPi

On RasPi change it in `sudo nano /etc/dhcpcd.conf`:
```
  static domain_name_servers=1.1.1.1 2606:4700:4700::1111 # exmaple for cloudflared
```
- if you do not specifiy a ipv4 and ipv6 address, the one not specified will still be the default one of the router
- `sudo service dhcpcd restart`
- resolvconf service is running on raspberry os which is creating /etc/resolv.conf dynamically from dhcpcd.conf entries
- You can check the changes in `sudo nano /etc/resolv.conf`, but do edit here since it gets overwritten at reboot

### Locals Problem over SSH
there is often a error coming up due to problems with the locals on the RasPi if connecting via ssh and performing some operations
remove error from `perl -e exit` by commenting out `AcceptEnv LANG LC_*` from `/etc/ssh/sshd_config`

### Create root user pw
I find it easier to access the RasPi directly via the root user, since all tasks that I perform on the RasPi are root tasks anyways, since I do only use it for server functionalities
- create root user password
```
sudo passwd
```

- remove the entry that the pi user is able to access root without needing a password
```
 sudo rm /etc/sudoers.d/010_pi-nopasswd
```

### Disable ssh login via pw
For security reasons it is good to have disabled the access via ssh over password prompt during the installation via the raspi imager or do now

```
sudo nano /etc/ssh/sshd_config
```

change lines

```
PasswordAuthentication no
```

### SSH connection via key
Now create an entry to be able to automatically connect to the root user via ssh from your PC

Start a root shell
```
sudo su
```
```
mkdir ~/.ssh/
chmod 700 ~/.ssh  # this is important. 
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys  #this is important.
nano ~/.ssh/authorized_keys
```

Then copy public key from your pc (.ssh/id_rsa.pub) to authorized_keys into the root user file on the Raspi you just opened
- display your key on your PC in a shell on your PC
```
gedit ~/.ssh/id_rsa.pub #show the key on your pc
```

Now you can disconnect from ssh and log in again into the root user
```
ssh root@your.ip.for.raspi
```

### Disable wifi
Disable wifi on reboot if you do not use it:

```
sudo crontab -e
@reboot (sleep 10 && ip link set wlan0 down)
```

### Set wifi password and SSID
If you want to use wifi
https://www.seeedstudio.com/blog/2021/01/25/three-methods-to-configure-raspberry-pi-wifi/

follow guide and change values in here
```
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

### Mounting external drive
If you have an external drive for data
```
blkid
```

```
mkdir /mnt/data
```

In
```
nano /etc/fstab
```

```

/dev/sda3 /mnt/data ext4 defaults,nofail 0 2 #change according to your device entries from blkid result
```

### Swapfile
Create a swapfile to have less problems with running out of RAM
see [Linux System (Ubuntu)#Swapfile]({{< ref "Linux System (Ubuntu)#swapfile" >}})

Sometime useful for RasPi if swap file is not in fstab
```
dphys-swapfile swap[on|off]
```

## More
Now we have a working Pi. Next step is to install services and do advanced setups

### Mail server
[mail server](https://www.lastbreach.de/blog/info-mails-fuer-unattended-upgrades)

```
sudo apt install msmtp
```

```
nano /etc/msmtprc
```

This is an example for my mail sever hosted at strato.de
```
# Defaults
defaults
auth           on
tls            on
tls_starttls   off
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        ~/.msmtp.log

# Configuration
account        strato            
host           smtp.strato.de   
port           465
from           mymail@domain.de
user           mymail@domain.de
password       mypw                  

# Standard Account for E-Mails
account default : strato     
```

- Try if it works via: `echo "This is a test" | msmtp -v somemail@something.de`
- Or via `echo -e "Subject: This is a subject \n\n This is a message \n\n" | sudo msmtp -v somemail@something.de`

Also link this setup to other executables, which might be used by some services
```
sudo chown root:root /etc/msmtprc
sudo chmod 600 /etc/msmtprc
sudo ln -s /usr/bin/msmtp /usr/sbin/sendmail
```

### Unatended Upgrades
Check this guide here: [security guide](https://raspberrytips.com/security-tips-raspberry-pi/)

Some basic steps from the guide:
```
sudo apt install unattended-upgrades
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

```
//Unattended-Upgrade::Mail "";
```

- Change more as you like...
- For mail notification you have to have a [#Mail server]({{< ref "#mail-server" >}}) installed
	- set mail server in config and also if on error or on change...

Also change this:
```
sudo nano /etc/apt/apt.conf.d/02periodic
```

and paste this content
```
APT::Periodic::Enable "1";
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "1";
APT::Periodic::Verbose "2";
```

or interactively
```
dpkg-reconfigure -plow unattended-upgrades
```

### Update scripts
I also have an update script located at my RasPis home folder, which I can always execute to update them regularly
- The script also contains parts to update my docker containers, more about this in the [Docker and Server Apps#Update Containers]({{< ref "Docker and Server Apps#update-containers" >}}) section

### Secuirity / Fail2Ban
[security guide](https://raspberrytips.com/security-tips-raspberry-pi/)

```
sudo apt install fail2ban
sudo nano /etc/fail2ban/jail.conf
sudo nano /etc/fail2ban/fail2ban.conf
sudo service fail2ban restart
```

### Tailscale VPN

Tailscale VPN can be used to easily connect to your local network based on wireshark protocol without the need for port forwarding. Tailscales server are used to enable this. So you need an account. Tough, I am really a fan of tailscale.

- [download](https://tailscale.com/download/linux/rpi-bullseye)
- [docs](https://tailscale.com/kb/installation/)

```
curl -fsSL https://tailscale.com/install.sh | sh
```

- settings are generated and stored under `/var/lib/tailscale`
- **Info**: Tailscale stores the info about last used startup flags and uses those for next up command, use --reset to use default values

#### Exit Node
```
# Up
sudo tailscale up --advertise-exit-node --advertise-routes=192.168.0.0/24
# Down
sudo tailscale down
```

#### Client
Also check out my Tailscale application launchers [from my linux-scripts repo](https://github.com/frederikb96/Linux-Scripts)

```
sudo tailscale up --reset --exit-node=100.73.113.18
# Or
sudo tailscale up --reset
# Down
sudo tailscale down
```
- If using an exit-node, all traffic is fully routed throught this device
- If only connecting to the tailscale network, only DNS settings (see next section [#Tailscale DNS]({{< ref "#tailscale-dns" >}})) are applied and only connections to other devices in the tailscale network are routed via the VPN
	- This has the benefit, that your connection is way faster, since the RasPi is normally the bottleneck regarding the internet speed
	- But it is also not as secure as routing all traffic through your RasPi if you are in a place, where the normal connection might not be secure

Status:
```
tailscale status --json --peers=false | grep Online
```

#### Tailscale DNS
You can also use tailscale to change the DNS server, the devices connected to tailscale are using
- I use tailscale to always be able to conntect to my Pi-hole local DNS server ([Docker and Server Apps#Pi-hole and Cloudflared]({{< ref "Docker and Server Apps#pi-hole-and-cloudflared" >}}))
- Check out this guide from [tailscale about Pi-hole](https://tailscale.com/kb/1114/pi-hole/?q=dns)
- You basically only have to have your RasPi, which is running your Pi-Hole connected via tailscale
- Then, visit the tailscale website and log in to your account and under settings and DNS, check the checkbox to overwrite local DNS settings
- Now specifiy your tailscale IP of your RasPi as the global nameserver
- Now your devices will use the Pi-Hole as their DNS server


#### Tailscale Alternative Installation via Docker

Docker installation not working somehow and it is not at all sandboxed within docker, since it needs full network mode anyway, so better use the real installation as above explained

[docker](https://hub.docker.com/r/tailscale/tailscale)

```
tailscale:
    container_name: tailscale
    image: tailscale/tailscale:latest
    hostname: pi3-tailscale
    volumes:
      - '/dev/net/tun:/dev/net/tun'
      - '/var/lib:/var/lib'
      - '/lib/modules:/lib/modules'
    network_mode: host
    privileged: true
    command: 'tailscaled'
    restart: always
```

Start with

```
docker exec -it tailscale tailscaled
docker exec -it tailscale tailscale status
docker exec -it tailscale tailscale up
```


## Network Setup
To make your Raspberry accessible from the outside world, read this page: [Network]({{< ref "Network" >}})

You do not need this, if you are the only one, who wants to access your servers. In this case use a VPN to connect to your local network, since this is much more secure.

## Install applications via Docker
See the next section for how to use docker to install server apps on your RasPi: [Docker and Server Apps]({{< ref "Docker and Server Apps" >}})

## Alternative OS Umbrel

Umbrel is an OS based on Raspberry OS, which is very user friendly and easy to setup: https://umbrel.com/
- List with available apps and their docker implementations for umbrel: https://github.com/getumbrel/umbrel-apps

Some basic setup steps after installation according to guide:
- login via ssh and do same steps as for normal OS, with ssh login and so on

**Problems:**
- You do not have that much control (configurations) over your installed docker apps
- Umbrel tries to do everything in your local network only
	- Access from outside shall only be possible via VPN
	- Thus, it is not so easy to use it as a publicly available server

I tried to configure it, such that I can use caddy to allow access from the outside world
- However, nginx was already present and was controlling port 80
- So caddy could not get access to this port
- I added a reverse proxy to nginx to pass traffic to caddy, but the config file gets overwritten with every umbrel update
	- `... server { listen 80; server_name *.bergrunde.net; location / { proxy_pass http://localhost:81; } } server { listen 80 default_server; ...`
	- Also this way it was not possible to use port 80 for ACME auto caddy challenge
- So for me, I decided to not use umbrel and do every docker installation on my own with a normal Raspberry OS