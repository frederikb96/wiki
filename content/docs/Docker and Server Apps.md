---
title: "Docker and Server Apps"
---
# Docker and Server Apps
The first part of this page is about Docker and my best practices. The second part is about the apps, which I have installed on my server. I install almost all my apps via docker.

## Docker
[Docker website](https://docs.docker.com)

### Install
You can install docker via the offical website or as packages from your package manager.
- I would recommend going with the [official website](https://docs.docker.com/engine/install/debian/).
- There are also small difference in the commands. `docker compose` vs `docker-compose` for example if using the one from your package manager.

**64 bit**
- I use a 64 bit installation for my RasPi and consequently also use the 64 bit installation of docker.
- This also has advantages, since some programs only run on 64 bit.
- Consequently use the 64 bit debian repo guide - use the convienience script, it is working for RasPi if you have the 64 bit version

Steps, manually:
- add repo
- install docker via repo
- check installation
- `docker run hello-world`
You can add users to docker group (`usermod -aG docker pi`), if users shall be able to execute docker without sudo. **Note:** This is not recommended, since this is dangerous, as normal users can get root access this way! Better execute all docker commands as root only!

### Docker compose vs Docker run
[Docker compose](https://docs.docker.com/get-started/08_using_compose/)
- Docker run is the command line version to start single containers via a command with flags added as arguments
- Docker compose uses a docker compose file (`docker-compose.yml`) where your docker configuration is configured and stored
- This has the advantage
  - to start multiple containers at once
  - to have all container configurations always stored
  - to be able to easily update the containers (stop, update, start)

[Converter to convert docker run command into docker compose script](https://www.composerize.com/)

I will only use docker compose scripts!

### My Docker Folder Structure
[Also read the getting started](https://docs.docker.com/get-started/)
- And the part about [docker volumes](https://docs.docker.com/get-started/05_persisting_data/)
  - These are used to store data in your containers between restarts of your containers
  - You can think about it like the hard drive of your PC
To manage all my programs as containers in a convenient way, I use the following file structure
- I store every data related to my docker containers in the folder `/opt/docker`
- Within this folder I create sub folders for different program groups, like a folder `/opt/docker/info` for system containers like [#Portainer]({{< ref "#portainer" >}}) and [#Docker Notify Diun]({{< ref "#docker-notify-diun" >}})
- Within those sub folders, I have a docker compose file, which stores the configuration for this container group
- In this folder are also further subfolders, which represent the volumes ("hard drives") of my containers
- This way all data related to these containers is stored within this folder
  - Except for some volume mounts, which are necessary if the container needs access to specific host OS folders; But these folders are normally only used to access data and not to write data
- If I want to transfer this program to another system, I simply have to tar the folder and untar it on the other system, all data is preserved and transferred this way :)

Example for a docker compose file, where we have one host OS folder mounted, which is needed for some functionality; But the other folder is the data folder of the container, where real data is stored between restarts and this is mounted to a subfolder:
```
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    ports:
      - 8000:8000
      - 9443:9443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer:/data
	restart: always
```

### Update Containers
You can put the following script into your home directory on your RasPi and execute it every time you want to update all containers by using the docker compose functionalities
- You can get notifications if new container updates are available, see further below for how to do that [#Docker Notify Diun]({{< ref "#docker-notify-diun" >}})
- The script will cd to the path where your docker compose files are located and then execute `docker compose pull` to get the latest images and `docker compose up -d` to update the containers; It will also display the container logs and you can use grep to filter the logs for specific keywords, like "error" or so
- It will also start the package updates

```
#!/bin/bash

# Docker update function
docker_update() {
echo ""
echo "###"
echo "Start docker update"
echo "###"
docker compose pull
docker compose up -d
sleep 5
docker compose logs -t --since 168h | grep -i err | grep -v -e "PHP_ERROR_LOG" -e "OCSP stapling"

}

# Main

# Docker
cd /opt/docker/dns
docker_update
cd /opt/docker/info
docker_update
cd /opt/docker/apps
docker_update

echo ""
echo "###"
echo "Docker Prune"
echo "###"
docker image prune -f

# APT
echo ""
echo "###"
echo "APT"
echo "###"
apt update && apt upgrade && apt autoremove
```

You can also use this script to restart all containers, if there are some problems during the update

```
#!/bin/bash

# Docker update function
docker_restart() {
echo ""
echo "###"
echo "Start docker restart"
echo "###"
docker compose down
docker compose up -d
sleep 1

}

# Main

# Docker
cd /opt/docker/dns
docker_restart
cd /opt/docker/info
docker_restart
cd /opt/docker/apps
docker_restart
```

### Volume Backups
#### Option 1
Just use tar to backup the container folders, as described on the [page about backups]({{< ref "Linux Backup#raspberry-pi-backups" >}}). This is what I do.

#### Option 2
Theoretically, it is better to let docker handle everything and only access the files in the way the docker container sees those, so more complicated but nicer and better if data is distributed somewhere on host file system. [Official guide](https://docs.docker.com/storage/volumes/#backup-restore-or-migrate-data-volumes)

Backup data example for vaultwarden and caddy:
```
docker run --rm --volumes-from vaultwarden -v /opt/docker/backup:/backup vaultwarden/server tar cvf /backup/backup_vaultwarden.tar /data

docker run --rm --volumes-from caddy -v /opt/docker/backup:/backup caddy:2 tar cvf /backup/backup_caddy.tar /data
```
- mounts volume of container vaultwarden
- additional mounts the current directory as backup folder in new container
- uses same docker image as the one to do the backup from
- executes command in new container: tar with where to place (in my mounted new volume) and the one to tar (the location of the old container which shall be backed up)

This script tries to automate this process:
```
#!/bin/bash
# This script allows you to backup a single volume from a container
# Data in given volume is saved in the current directory in a tar archive.
CONTAINER_NAME=$1
VOLUME_NAME=$2

usage() {
  echo "Usage (backup will be created in current folder): $0 [container name] [volume name]"
  exit 1
}

if [ -z $CONTAINER_NAME ]
then
  echo "Error: missing container name parameter."
  usage
fi

if [ -z $VOLUME_NAME ]
then
  echo "Error: missing volume name parameter."
  usage
fi

echo ""
echo "### Backup started for $CONTAINER_NAME : $VOLUME_NAME ###"
echo ""
sudo docker run --rm --volumes-from $CONTAINER_NAME -v $(pwd):/backup busybox tar cvf /backup/backup.tar $VOLUME_NAME
```

This script tries to automate the process of restoring a backup:
```
#!/bin/bash
# This script allows you to restore a single volume from a container
# Data in restored in volume with same backupped path
NEW_CONTAINER_NAME=$1
BACKUP_NAME=$2

usage() {
  echo "Usage only from within the folder containing the backup: $0 [container name] [backup name]"
  exit 1
}

if [ -z $NEW_CONTAINER_NAME ]
then
  echo "Error: missing container name parameter."
  usage
fi

if [ -z $BACKUP_NAME ]
then
  echo "Error: missing backup name parameter."
  usage
fi

sudo docker run --rm --volumes-from $NEW_CONTAINER_NAME -v $(pwd):/backup busybox tar xvf /backup/$BACKUP_NAME
```

This script uses the previous backup scripts to do multiple backups of specific containers:
```
#!/bin/bash
# Backup all my containers

# ./docker_backup.sh container_name container_folder
# mv -f backup.tar backup_container_name_container_folder.tar

cd /opt/docker/backup

# Portainer
./docker_backup.sh portainer data
mv -f backup.tar backup_portainer.tar
# Vaultwarden
./docker_backup.sh vaultwarden data
mv -f backup.tar backup_vaultwarden.tar
# Caddy
./docker_backup.sh caddy etc/caddy/Caddyfile
mv -f backup.tar backup_caddy_file.tar
./docker_backup.sh caddy config
mv -f backup.tar backup_caddy_config.tar
./docker_backup.sh caddy data
mv -f backup.tar backup_caddy_data.tar
# Pi-hole
./docker_backup.sh Pi-hole etc/Pi-hole
mv -f backup.tar backup_Pi-hole_etc.tar
./docker_backup.sh Pi-hole etc/dnsmasq.d
mv -f backup.tar backup_Pi-hole_dnsmasq.tar
# Diun
./docker_backup.sh diun etc/diun/diun.yml
mv -f backup.tar backup_diun_yml.tar
./docker_backup.sh diun data
mv -f backup.tar backup_diun_data.tar
```

### My Docker Network Structure
- I have many containers, which are somehow accessing the internet or providing sockets to be accessed via the Internet (like Nextcloud and so on)
  - To manage the access between containers and the internet, I use one single entry point, my caddy (also see the [caddy guide later]({{< ref "#caddy" >}})) web server, which is used as a reverse proxy to forward all queries to the correct containers
- The different containers also have to communicate with each other

To solve these two problems, I use the docker network functionality
- You can read about this in the official documents, [container communication via bridges](https://docs.docker.com/network/bridge/) and [networking in docker compose files](https://docs.docker.com/compose/networking/)
- Containers can be part of multiple networks

#### Container Networks
If two containers have to interact with each other, I specify a network within the docker compose file. E.g. if one container needs to have access to another data base container, both are getting the same network:
```
...
mariadb:
  ...
  networks:
    - db
...

nextcloud:
  ...
  networks:
    - db
...

networks:
  db:
```

And then there is one special case, which is my caddy reverse proxy server. Since I have plenty of different containers started within different docker compose files, I create an external network once manually, where every container will be part of, that wants to be accessible from the internet via some domain.
- Create the caddy network manually via a command:
  - `docker network create --subnet=172.90.0.0/16 caddy`
  - I also specified a subnet, which is sometimes necessary, if some containers need a static IP address (is interesting later for some containers)
- And now I can specify within the docker compose files, that these containers also want to be part of this network
```
...
mariadb:
  ...
  networks:
    - db
...

nextcloud:
  ...
  networks:
    - db
    - caddy
...

networks:
  db:
  caddy:
    external: true
```

#### Reverse Proxy
Normally containers would open some ports to the host OS, such that they are accessible from the internet, like this:
```
...
nextcloud:
  ...
  ports:
    - 80:80
    - 443:443
...

```

However, the only container with open ports that I have, is my caddy container. Since the caddy container is in the same network as e.g. the Nextcloud container, it can still directly access the Nextcloud container, without the need of open ports. Consequently, it can forward traffic arriving for a specific domain, directly to the Nextcloud container on e.g. port 80. Check out the [guide on caddy]({{< ref "#caddy" >}}) further below. To get an understanding, a caddy file would look like this:
```
https://cloud.my-domain.de:443 {
	# Use the ACME HTTP-01 challenge to get a cert for the configured domain.
	tls {$EMAIL}

	rewrite /.well-known/carddav /remote.php/dav
	rewrite /.well-known/caldav /remote.php/dav

	reverse_proxy nextcloud:80
}

```
We can see, that I can directly forward all traffic arriving for `cloud.my-domain.de` to the container named `nextcloud` on port 80

There are also containers, which shall only be accessible in my local network. Check out the [guide about Pi-hole]({{< ref "#pi-hole-and-cloudflared" >}}) how I use local DNS to access those containers nicely. Furthermore, I also use reverse proxies for those containers. This would look like this (but also check the [caddy guide]({{< ref "#caddy" >}})):
```
https://portainer.pi4.lan:443 {
	tls internal
	reverse_proxy portainer:9000
}
```

#### Other Network Information
Some information, I often forget.

How to create a network within docker compose and specify the subnet:
```
networks:
  my-network:
    ipam:
      config:
       - subnet: 172.25.0.0/16
```

### Docker Commands
- `docker compose up -d`
- `docker compose down`
- `docker compose logs -f` Display logs since start of container
- `docker compose pull`

Commands in container
- `docker exec -it diun ls` Execute a command within the container
- `docker exec -it diun /bin/bash` Open a command shell on container

## Apps
- Tips for interesting apps for RasPi: https://github.com/awesome-selfhosted/awesome-selfhosted
- Some interesting apps with docker compose templates https://github.com/getumbrel/umbrel-apps

This section documents the apps, that I run on my RasPi servers via docker
- **Note:** All containers, which rely on being accessible from the internet, are part of the caddy network and consequently the presented compose files only work if you have the same setup as I described in [#My Docker Network Structure]({{< ref "#my-docker-network-structure" >}}).
- I also add a small snippet of the relevant part from the caddy file to every app, such that you can easily set up your reverse proxies

### Caddy
[Caddy](https://caddyserver.com/docs/getting-started) is the webserver, that I use to redirect all internet queries via reverse proxies. You can also use nginx or Apache, but I prefer caddy, since it is very user-friendly (it automatically handles ssl certificates via letsencrypt) and is very well documented.

Extract of docker compose file:
```
services:

  caddy:
    image: caddy:2
    container_name: caddy
    ports:
      - 80:80  # Needed for the ACME HTTP-01 challenge.
      - 443:443
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
      - ./caddy/caddy-config:/config
      - ./caddy/caddy-data:/data
    environment:
      EMAIL: "your-email@provided.de" # The email address to use for ACME registration.
    networks:
      caddy:
        ipv4_address: 172.90.0.100
    restart: always
   
networks:
  caddy:
    external: true
```
- I give the caddy server a static IP within the external caddy network, since it is sometimes necessary to know the IP address of the reverse proxy server within other containers, that rely on this reverse proxy server

#### Caddyfile
The caddy file defines the setup of your webserver
- You can specify how specific queries are handled
- I mainly use it for reverse proxies (to forward the traffic)
	- I have some domains, which shall be accessible from the internet
		- Make sure to have read the page about [how to make your server globally accessible]({{< ref "Network#server-global-accessible" >}})
		- If you fulfilled all the requirements, you can simply use caddy to automatically handle the https certificate process for you as described in the [caddy guide about https](https://caddyserver.com/docs/automatic-https)
		- In short
	- You have to have a domain pointing to your IP address
		- Your router must have open ports 80 and 443 for your server
		- In your caddy file you have to specify that you want to automatically let caddy handle everything, example:
```
https://your-comain.de:443 {
	# Use the ACME HTTP-01 challenge to get a cert for the configured domain.
	tls {$EMAIL}

	respond "Welcome to my website."
}
```
- .
	- .
		- That is all, now you will receive an email, if the generation was successful
		- You can also check the docker logs, if errors occurred `docker compose logs`
		- This is most likely because your server is inaccessible from the internet (ports not open or domain not pointing to your server), check [Network#Server Global Accessible]({{< ref "Network#server-global-accessible" >}}) again in this case
	- If you have domains, which shall only be accessible from the local network
		- You can use caddy to create self-signed certificates, like this:
```
https://pi4.lan:443 {
	tls internal
	respond "Welcome to my local only website."
}
```
- .
	- .
		- Now you also have an encrypted connection to your website, which is only accessible from the local network
		- **Note:** This works only if you have a local dns resolver like [#Pi-hole and Cloudflared]({{< ref "#pi-hole-and-cloudflared" >}}), which resolves something like `pi4.lan` for you. If that is not the case, you can simply replace the domain within the caddy file, with the static ipv4 address of your server. This is also fine :)
- I also have a simple caddy setup to server some static sites (also see [#Hugo]({{< ref "#hugo" >}}) which is my static site generator)
	- You can put your static sites for example into the caddy data vault `caddy/caddy-data/websites/your-files`, which will be the following path within the caddy container `/data/websites/your-files`
```
https://website.my-domain.de:443 {
	# Use the ACME HTTP-01 challenge to get a cert for the configured domain.
	tls {$EMAIL}
	
	root * /data/websites/your-files
	file_server
}
```

On example how a caddy file could look like later, if you have multiple containers running that need to be accessible from the internet:

```
https://cloud.my-domain.de:443 {
	# Use the ACME HTTP-01 challenge to get a cert for the configured domain.
	tls {$EMAIL}

	rewrite /.well-known/carddav /remote.php/dav
	rewrite /.well-known/caldav /remote.php/dav

	reverse_proxy nextcloud:80
}

https://my-domain.de:443 {
	# Use the ACME HTTP-01 challenge to get a cert for the configured domain.
	tls {$EMAIL}

	respond "Welcome to my website"
}

https://website.my-domain.de:443 {
	# Use the ACME HTTP-01 challenge to get a cert for the configured domain.
	tls {$EMAIL}
	
	root * /data/websites/your-files
	file_server
}

https://php.lan:443 {
	tls internal
	reverse_proxy phpmyadmin:80
}

https://portainer.pi4.lan:443 {
	tls internal
	reverse_proxy portainer:9000
}
```

**Reload caddy file**
```
docker exec -ti caddy caddy reload --config /etc/caddy/Caddyfile
```
- **Not working** within docker container, somehow...; I need to `docker restart caddy` to make changes visible

**Caddy directives tags for caddyfile**
https://caddyserver.com/docs/caddyfile/directives

### Docker Notify Diun
Checks for new versions on all containers on docker registry and get a notification by mail
- See also [diun](https://crazymax.dev/diun/) with [basic example](https://crazymax.dev/diun/usage/basic-example/) and guide on [config](https://crazymax.dev/diun/config/)
```
services:
  diun:
    image: crazymax/diun:latest
    container_name: diun
    command: serve
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./diun/data:/data"
      - "./diun/diun.yml:/etc/diun/diun.yml"
    environment:
      - "TZ=Europe/Berlin"
      - "LOG_LEVEL=info"
      - "LOG_JSON=false"
      - "DIUN_WATCH_WORKERS=20"
      - "DIUN_WATCH_SCHEDULE=0 0 16 * * *"
      - "DIUN_PROVIDERS_DOCKER=true"
      - "DIUN_PROVIDERS_DOCKER_WATCHBYDEFAULT=true"
    dns:
      - 1.1.1.1
    restart: always
    
networks:
  caddy:
    external: true
```

and config file in diun/data

```
notif:
  mail:
    host: smtp.strato.de
    port: 465
    ssl: true
    insecureSkipVerify: false
    username: webmaster@your-domain.de
    password: pw
    from: webmaster@your-domain.de
    to:
      - your-mail@provider.de
    templateTitle: "{{ .Entry.Image }} released for pi4"
    templateBody: |
      Docker tag {{ .Entry.Image }} which you subscribed to through {{ .Entry.Provider }} provider has been released.

```
### Portainer
You can use [portainer](https://hub.docker.com/r/portainer/portainer-ce/) to manage docker via GUI [wiki](https://docs.portainer.io/v/ce-2.9/start/install/server/docker/linux)\:
```
services:
  
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    #ports:
      #- 8000:8000 #necessary for agent, if connect from other device
      #- 9000:9000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer/portainer_data:/data
    networks:
      - caddy
    
networks:
  caddy:
    external: true
```

### Pi-hole and Cloudflared
Pi-hole can be used as a local DNS service, which has benefits like encrypting your DNS queries via Cloudflare and blocking ads in the web
- [Website Pi-hole](https://pi-hole.net/)
  - We use [Pi-hole docker](https://hub.docker.com/r/Pi-hole/Pi-hole/)
    - [Pi-hole wiki for docker](https://github.com/pi-hole/docker-pi-hole)
- [Cloudflare official guide, but no RasPi (arm) support](https://hub.docker.com/r/cloudflare/cloudflared)
  - [Cloudflare official guide for DNS over https](https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/)
- So we use [this custom docker image for arm](https://hub.docker.com/r/visibilityspots/cloudflared)
  - You can also check out [this guide](https://visibilityspots.org/dockerized-cloudflared-pi-hole.html)

Docker compose file:
```
services:
 
  cloudflared:
    container_name: cloudflared
    image: visibilityspots/cloudflared:latest
    environment:
      - 'ADDRESS=::'
    dns:
      - 1.1.1.1
    networks:
      - dns
    restart: always
    
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    # For DHCP it is recommended to remove these ports and instead add: network_mode: "host"
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      #- "67:67/udp" # Only required if you are using Pi-hole as your DHCP server
      #- "8080:80/tcp" #Proxy pass
    environment:
      TZ: 'Europe/Berlin'
      WEBPASSWORD: 'pw'
      FTLCONF_LOCAL_IPV4: '<local ipv4 address of your RasPi>'
      PIHOLE_DNS_: 'cloudflared#5054'
      VIRTUAL_HOST: 'pihole.lan'
    # Volumes store your data between container upgrades
    volumes:
      - './pihole/etc-pihole:/etc/pihole'
      - './pihole/etc-dnsmasq.d:/etc/dnsmasq.d'    
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    #cap_add:
      #- NET_ADMIN # Required if you are using Pi-hole as your DHCP server, else not needed
    networks:
      - dns
      - caddy
    depends_on:
      - cloudflared
    restart: always
    
networks:
  dns:
  caddy:
    external: true

```
- We have to open port 53, such that the DNS server can be accessed
- Port 80 is not necessary, since we use caddy again to access the web view of our Pi-hole via reverse proxy Caddy file:
```
https://<local ipv4 address of your RasPi>:443 {
  tls internal
  reverse_proxy pihole:80
}
```
- Now you can connect to the Pi-hole web interface
  - Check under settings and DNS that your upstream server is correctly set
    - We want to use cloudflared and `cloudlfared#5054` should be automatically resolved to the IP address of the docker container
      - If there are problems, use a hard-coded IP address, which you have to specify in the docker compose file
  - **Local DNS**\: If you would like to create domains, which should be resolved to IP addresses in your local network, you can specify those via this setting
    - I use it to be able to also access my local services via domains and this way I can also use caddy as a reverse proxy for these domains to forward the queries to the correct containers
    - So e.g. I add an entry like `pi3.lan` represents `192.168.0.203` (IP of my RasPi3) and so on
- Now you also need to configure your devices to use your new Pi-hole as a DNS server, see [Network#Change DNS]({{< ref "Network#change-dns" >}}) for this
- If your device is using your new Pi-hole as a DNS server, you can also replace the `<local ipv4 address of your RasPi>` with your new local DNS (e.g. `pihole.lan`), which you might have set up, in your Caddy file

### Bitwarden / Vaultwarden
Vaultwarden is Bitwarden for low resource servers like a RasPi. It can be used as a password manager

- [Bitwarden docu](https://bitwarden.com/help/install-on-premise-linux/)
- [Difference to Vaultwarden](https://www.reddit.com/r/selfhosted/comments/p54no4/vaultwarden_vs_official_bitwarden_server/)
- [Vaultwarden docker site](https://hub.docker.com/r/vaultwarden/server)
  - [Wiki of vaultwarden](https://github.com/dani-garcia/vaultwarden/wiki)
  - [Vaultwarden and caddy](https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose)
    - [Vaultwarden https](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS#enabling-https)

Docker compose script:
```
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    environment:
      DOMAIN: "https://vaultwarden.my-domain.de"
      WEBSOCKET_ENABLED: "true"  # Enable WebSocket notifications.
      SIGNUPS_ALLOWED: "false"
      SMTP_HOST: "smtp.strato.de"
      SMTP_FROM: "webmaster@my-domain.de"
      SMTP_FROM_Name: "Vaultwarden"
      SMTP_PORT: 465
      SMTP_SECURITY: "force_tls"
      SMTP_USERNAME: "webmaster@my-domain.de"
      SMTP_PASSWORD: "pw"
      #ADMIN_TOKEN: "pw"
    volumes:
      - ./vaultwarden/vw-data:/data
    networks:
      - caddy
    restart: always
    
networks:
  caddy:
    external: true
```

Caddy file:
```
https://vaultwarden.your-domain.de:443 {
  log {
    level INFO
    output file /data/access.log {
      roll_size 10MB
      roll_keep 10
    }
  }

  # Use the ACME HTTP-01 challenge to get a cert for the configured domain.
  tls {$EMAIL}

  # This setting may have compatibility issues with some browsers
  # (e.g., attachment downloading on Firefox). Try disabling this
  # if you encounter issues.
  encode gzip

  # Notifications redirected to the WebSocket server
  reverse_proxy /notifications/hub vaultwarden:3012

  # Proxy everything else to Rocket
  reverse_proxy vaultwarden:80 {
       # Send the true remote IP to Rocket, so that vaultwarden can put this in the
       # log, so that fail2ban can ban the correct IP.
       header_up X-Real-IP {remote_host}
  }
}
```

### WireGuard
**Better use tailscale** [Raspberry Pi#Tailscale VPN]({{< ref "Raspberry Pi#tailscale-vpn" >}})
- Problems with ipv4 connection via smartphone and not very easy to connect to - also no DNS only function ...
- https://www.wireguard.com/
- https://github.com/linuxserver/docker-wireguard
- https://www.procustodibus.com/blog/2021/01/wireguard-endpoints-and-ip-addresses/

```
  wireguard:
    image: linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=0
      - PGID=0
      - TZ=Europe/Berlin
      - SERVERURL=vpn.my-domain.de #optional
      - SERVERPORT=51820 #optional
      - PEERS=smartphone pc another_device #optional
      #- PEERDNS=auto #optional
      #- ALLOWEDIPS=0.0.0.0/0 #optional
      - LOG_CONFS=false #optional
    volumes:
      - ./wireguard/config:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    restart: always
```
- For new peers change the peer variable; You can also delete the peer directories to force recreation

#### Ubuntu Connection
Guide to connect via Ubuntu to Wireguard VPN

Symlink to fix dependencies:
```
ln -s /usr/bin/resolvectl /usr/local/bin/resolvconf
```

Then copy config to `/opt/wireguard/` and start:
```
sudo wg-quick up /opt/wireguard/bergrunde.conf
sudo wg-quick down /opt/wireguard/bergrunde.conf
```

### Nextcloud
- [Manual Nextcloud](https://docs.nextcloud.com/server/latest/admin_manual/)
- [Docker](https://github.com/docker-library/docs/blob/master/nextcloud/README.md)
- [Tutorial for Caddy](https://gist.github.com/tmo1/72a9dc98b0b6b75f7e4ec336cdc399e1)

#### Setup via Docker Compose
An overview over main "components":
- Containers:
	- MariaDB as a database
	- Redis
	- Nextcloud main container
	- Nextcloud cron container
		- Used for sync jobs
- Data folder
	- I use an external drive, which I mount at `/mnt/data` of my RasPi
	- This folder gets then mounted as a vault, to be used as the data folder for Nextcloud
- config.php file for all configurations
- Apache server which is auto started within container
- Reverse proxy via caddy
- Optional: phpmyadmin to check data base

```
services:
  
  mariadb:
    image: mariadb
    container_name: mariadb
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - ./mariadb/mysql:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=pw_sql_root
      - MYSQL_PASSWORD=pw_sql_next
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    networks:
      - nextcloud
      - db
    restart: always

  redis:
    image: redis
    container_name: redis
    command: redis-server --requirepass "pw_red"
    volumes:
      - ./redis/data:/data
    networks:
      - nextcloud
    restart: always

  nextcloud:
    image: nextcloud
    container_name: nextcloud
    volumes:
      - ./nextcloud:/var/www/html
      - /mnt/data/nextcloud:/nextcloud_data
    environment:
      - MYSQL_HOST=mariadb
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=pw_sql_next
      - MYSQL_DATABASE=nextcloud
      - REDIS_HOST=redis
      - REDIS_HOST_PASSWORD=pw_red
      - NEXTCLOUD_ADMIN_USER=admin
      - NEXTCLOUD_ADMIN_PASSWORD=pw_next_admin
      - NEXTCLOUD_TRUSTED_DOMAINS=cloud.your-domain.de
      - NEXTCLOUD_DATA_DIR=/nextcloud_data
    depends_on:
      - mariadb
      - redis
    networks:
      - nextcloud
      - caddy
    restart: always
    
  nextcloud_cron:
    image: nextcloud
    container_name: nextcloud_cron
    volumes:
      - ./nextcloud:/var/www/html
      - /mnt/data/nextcloud:/nextcloud_data
    entrypoint: /cron.sh
    depends_on:
      - mariadb
      - redis
    networks:
      - nextcloud
    restart: always
    
  phpmyadmin:
    image: phpmyadmin
    container_name: phpmyadmin
    restart: always
    #ports:
      #- 8081:80
    networks:
       - caddy
       - db
    environment:
      - PMA_HOST=mariadb
 
networks:
  caddy:
    external: true
  nextcloud:
  db:

```

And set up your caddy file:
```
https://cloud.your-domain.de:443 {
	# Use the ACME HTTP-01 challenge to get a cert for the configured domain.
	tls {$EMAIL}

	rewrite /.well-known/carddav /remote.php/dav
	rewrite /.well-known/caldav /remote.php/dav

	reverse_proxy nextcloud:80
}

https://php.lan:443 {
	tls internal
	reverse_proxy phpmyadmin:80
}
```

#### Post setup
- User of your data folder needs to be `www-data:www-data` and also no read/write access for others
- Once the containers are running, you can try to visit the website and log in with your admin account
- **Note:** All configuration can be done via the admin webinterface or via the `nextcloudvault/config/config.php` file. Also read the Nextcloud manual!
	- If the containers are started for the first time, the `config.php` file will be created, containing the settings, you specified in the environment variables of your Nextcloud container
	- After that, the environment variables will never overwrite the `config.php` file again, so any configuration cannot be done via the environment variables anymore
	-  fix reverse proxy warning by adding to the config:
		- `'trusted_proxies' => '<ip of caddy container>', 'overwritehost' => 'cloud.my-domain.de', 'overwriteprotocol' => 'https',`
	- setup mail account via admin panel or config
- Check in admin settings, that cron job is selected
- Install apps tasks, calendar, contacts, and everything else you want

#### OCC Commands
https://docs.nextcloud.com/server/20/admin_manual/configuration_server/occ_command.html
```
# Shell
docker exec -u 33 -it nextcloud bash

# Scan
docker exec -u 33 -it nextcloud php occ files:scan --all --verbose
docker exec -u 33 -it nextcloud php occ files:scan --path="/User/files/Music" --verbose

# Maintenance
docker exec -u 33 -it nextcloud php occ maintenance:mode --on
docker exec -u 33 -it nextcloud php occ maintenance:mode --off
```

#### Other
Mysql cheat sheet:
https://www.mysqltutorial.org/mysql-cheat-sheet.aspx 

### Collabora Online
Can be used in connection with your Nextcloud to edit documents online
- Needs many resources like RAM, RasPi is not enough for this

**Not fully tested**
```
  collabora:
    image: collabora/code
    container_name: collabora
    restart: always
    environment:
      username: admin
      password: pw
      domain: "office.my-domain.de"
      dictionaries: "de en"
    cap_add:
      - MKNOD
```

### Wiki.js
Wiki that can be selfhosted
[wiki js](https://docs.requarks.io/install/docker)

```
  postgres:
    image: postgres:11-alpine
    container_name: postgres
    environment:
      POSTGRES_DB: wiki
      POSTGRES_PASSWORD: pw_post
      POSTGRES_USER: wikijs
    logging:
      driver: "none"
    networks:
      - wiki
    volumes:
      - ./postgres:/var/lib/postgresql/data
    restart: always

  wiki:
    image: requarks/wiki:2
    container_name: wiki
    depends_on:
      - postgres
    environment:
      DB_TYPE: postgres
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: wikijs
      DB_PASS: pw_post
      DB_NAME: wiki
    #ports:
    #  - "8089:3000"
    networks:
      - wiki
      - caddy
    restart: always
 
networks:
  caddy:
    external: true
  wiki:
```

Caddyfile:
```
https://wiki.bergrunde.net:443 {
	# Use the ACME HTTP-01 challenge to get a cert for the configured domain.
	tls {$EMAIL}

	reverse_proxy wiki:3000
}
```

### Syncthing
Can be used to sync stuff
- https://syncthing.net/
- I do not use it though
- I use my backup methods as presented under [Linux Backup]({{< ref "Linux Backup" >}})

```
syncthing:
    image: linuxserver/syncthing:latest
    container_name: syncthing
    hostname: pi4-syncthing
    environment:
      - PUID=0
      - PGID=0
      - TZ=Europe/Berlin
    volumes:
      - /opt/syncthing:/config
      - /opt/test:/data1
    ports:
      #- 8384:8384
      - 22000:22000/tcp
      - 22000:22000/udp
      - 21027:21027/udp
    networks:
      - caddy
    restart: always
```

### Hugo
A tool to create static websites
- [Hugo website](https://gohugo.io/)
- [Hugo docker image, unofficial](https://hub.docker.com/r/klakegg/hugo)

**Note:** Hugo has an integrated webserver, which can be used to immediately view the files during development - we use a local domain address (`wiki.lan`) (see [#Pi-hole and Cloudflared]({{< ref "#pi-hole-and-cloudflared" >}})) to reverse proxy it via caddy. And then you can generate html code and publish it via your real webserver - we use caddy to directly publish the sites via a global domain (`your-domain.de`).

Docker compose file:
```
services:
  hugo:
    image: klakegg/hugo:ext-ubuntu
    container_name: hugo
    command: server --appendPort "false" -b "https://wiki.lan" -p 443
    volumes:
      - "./hugo:/src"
    networks:
      - caddy
    restart: always

networks:
  caddy:
    external: true
```

Caddy file - local:
```
https://wiki.lan:443 {
	tls internal
	reverse_proxy hugo:443
}
```

- the server will fail to start and the container will crash, if there is not a `config.toml` located inside the `/src` folder
	- consequently add the `config.toml` file with the following content to your volume `./hugo` before starting the container
```
baseURL = 'https://your-domain.de/'
languageCode = 'en-us'
title = 'My New Hugo Site'
```

- Now you can connect to the shell of the docker container
	- `docker exec -ti hugo bash`
- From now on you can use all the `hugo` commands as described in the [hugo guide](https://gohugo.io/getting-started/quick-start/)
	- If you go with the quickstart guide, you can create the quickstart folder within the /src folder and afterwards move all content one layer higher (and overwrite the original config.toml) and remove the empty quickstart folder afterwards
	- Skip the step with enabling the server, since it is always running anyways and watching this folder for updates
	- After a theme is inserted, you can visit your website and see the results the first time
- **Note:** The default hugo server (with pid 1) must always run within the container, which is started with the initial command in the docker compose file
	- If you stop the server, the container crashes
	- You can change the flags for this server in the docker compose file
	- With the current setup it is listening on port 443 and by default is always watching the `/src` folder for changes
	- I had problem if not using the same port as you will use when reverse proxying this server via caddy, since internal links will brake then
		- so I directly use port 443, which is ok, since caddy can directly forward to this port
		- The port is not published to the host network and only containers within the caddy network can access the hugo server via 443

Now we also want to publish the content to the real server:
- Simply call the `hugo` command within your docker container
	- First to a delete of the old files, if there are some: `docker exec -ti hugo rm -rf public`
	- `docker exec -ti hugo hugo`
- This will create the `public/` folder with the static website
	- Check `docker exec -ti hugo ls -la`
- Now you can paste the public folder to your webserver data folder (see [here how to server static files]({{< ref "#caddy" >}})) and serve it
Caddyfile:
```
https://my-domain.de:443 {
	# Use the ACME HTTP-01 challenge to get a cert for the configured domain.
	tls {$EMAIL}
	
	root * /data/websites/public
	file_server
}
```

#### Hugo Guide
This [video guide about Hugo](https://www.youtube.com/playlist?list=PLLAZ4kZ9dFpOnyRlyS-liKL5ReHDcj4G3) is very useful to understand the main concepts of the different directories located within the Hugo folder and how templates and themes work
- Most useful were the videos 5, 11, 12 and 13

#### Hugo Theme: Book
You can use the [Hugo theme book](https://themes.gohugo.io/themes/hugo-book/) for your website
- Follow the installation instructions
- Note, that there is currently a [bug reported for this image](https://github.com/alex-shpak/hugo-book/issues/471#issuecomment-1198279113)
	- You might have to delete two files to make the theme work

#### Obsidian to Hugo
- I want to export my Obsidian vault to Hugo
- You can use this script here for [obsidian-to-hugo conversion](https://github.com/devidw/obsidian-to-hugo)
- Basically:
	- `pip3 install obsidian-to-hugo`
	- `python3 -m obsidian_to_hugo --obsidian-vault-dir=<path> --hugo-content-dir=<path>`

##### Automatic Hugo Update
I want to have my Obsidian vault locally and if I make changes, they should automatically be published in my Wiki
- I use a git repository to accomplish this
	- The git repo will hold the complete hugo folder

###### Local Side
- My folder containing the vault, that shall be published is called `public/`
- I add another folder named `public-repo` next to it
- In the `public-repo` folder, I have some scripts

ini-repo.sh
- This script basically initializes my repo
	- You have to manually set up this repo, such that your complete hugo folder will be inside the repo
```
#!/bin/bash
git clone --recursive git@github.com:frederikb96/wiki.git ./local.wiki
echo "done"
read
```

update-repo.sh
- This script automatically pulls and simply commits all new changes that I make in the local repo
```
#!/bin/bash
cd local.wiki
git pull --no-rebase
git stage --all
git commit -m 'Auto update'
git push
echo "done"
```

convert-content.sh
- This script, uses the [#Obsidian to Hugo]({{< ref "#obsidian-to-hugo" >}}) method to convert my vault content and add it to the content folder of the hugo repo
```
#!/bin/bash
python3 -m obsidian_to_hugo --obsidian-vault-dir=../public --hugo-content-dir=local.wiki/content
```

all.sh
- This script simply calls the convert and afterwards the update script
```
#!/bin/bash
./update-repo.sh
./convert-content.sh
./update-repo.sh
echo "all done"
```

This way all your local vault content is always up2date with the git repo, you simply have to call the all.sh script every time you make a change, that shall be visible in the git repo

###### Server Side
Now we have to always be able to get updates on the server side
- Go to your docker hugo folder and initalize the hugo vault as your hugo git repository (basically remove all content and replace it with the git repo, which has to contain the same content)
	- Basically: `rm -rf hugo`  and `git clone --recursive https://github.com/frederikb96/wiki.git ./hugo`
	- Note: If your repository is public, you can pull via the "https" link, such that you do not have to add your git ssh key to your server, which is enough if you only want to pull and not commit anything via the server side
- Now you can always use a simple script to update your hugo repo on the server
	- In my setup, the script is located next do the `docker-compose.yml` file and is updating the hugo vault/repo
	- **Note:** It is only used to pull and not commit changes, so all local changes are always overwritten!
- We also want to update the real website and not only the local website
	- So we also automate the publishing process
	- And move the directory to the location where caddy is serving the new files

update-repo.sh
```
#!/bin/bash
cd hugo
git reset --hard
git pull --no-rebase
echo "done"
```

all.sh
```
#!/bin/bash
./update-repo.sh
rm -rf hugo/public
docker exec -ti hugo hugo && rsync -av --delete hugo/public/ /opt/docker/caddy/caddy/caddy-data/websites/hugo-wiki/ && rm -rf hugo/public
echo "all done"
```

- You can also setup a crontab job to automatically update your git repo every hour or so:
- `crontab -e`
- `0 * * * * (cd /opt/docker/hugo && ./all.sh)`

###### Both Sides
If you want to make changes directly visible, you can simply combine both previous sides
- You can also create launchers for this
```
[Desktop Entry]
Version=1.1
Type=Application
Name=Wiki Publish
Icon=
Exec=gnome-terminal -- bash -c 'cd /home/freddy/Nextcloud/Notes/Technical/public-repo && ./all.sh && ssh root@pi4.lan "(cd /opt/docker/hugo && ./all.sh)"; read'
Terminal=false
```

### XWiki
https://www.xwiki.org/xwiki/bin/view/Main/WebHome

Docker compose file:
```
services:
    
  xwiki:
    image: "arm64v8/xwiki"
    container_name: xwiki
    depends_on:
      - xwiki-db
    ports:
      - "8080:8080"
    environment:
      - DB_USER=xwiki
      - DB_PASSWORD=xwiki
      - DB_HOST=xwiki-db
    volumes:
      - ./xwiki:/usr/local/xwiki
    networks:
      - xwiki-db
      - caddy
      
  xwiki-db:
    image: "arm64v8/mysql"
    container_name: xwiki-db
    volumes:
      - ./db/xwiki.cnf:/etc/mysql/conf.d/xwiki.cnf
      - ./db/data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      - MYSQL_ROOT_PASSWORD=xwiki
      - MYSQL_USER=xwiki
      - MYSQL_PASSWORD=xwiki
      - MYSQL_DATABASE=xwiki
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_bin --explicit-defaults-for-timestamp=1 --default-authentication-plugin=mysql_native_password
    networks:
      - xwiki-db
 
networks:
  caddy:
    external: true
  xwiki-db:
```