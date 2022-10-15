---
title: "Network"
date: 2022-10-15
draft: false
---

# Network

## Overview Global Network
- There is also the local and the global network
	- IP addresses in your local network can normally be set to be stable, since you are controlling the local network
	- IP addresses in the global network are often dynamic, if you do not have an interner provider which is providing you a static ip address
- There are ipv4 and ipv6 addresses
	- ipv4 addresses are normally not provided to you anymore by your internet provider, though some still do so
		- Back in those days, your router got an ipv4 address and was forwarding requests via ipv4 to your devices in the local network
	- Today, your internet provider provides you with a ipv6 addresses for every device
		- So your devices are all theoretically directly addressable from the outside world
		- However, your router still blocks all queries by default for security reasons
		- If you want to make your server accessible, you have to still configure to forward specific traffic to your server
		- see more about this in the next section: [#Router Port Forwarding]({{< ref "#router-port-forwarding" >}})

Small excursion about dual stack light
- (look up dual stack light in the internet to get more information)
- Providers like vodafone, only give you an ipv6 address
- You can still connect as a client to other servers that only support ipv4, since vodafone is tunneling this traffic and is converting it to ipv4
	- this is what is happening for all the users every day when they browse the internet and so on (their PC's are always clients)
- However, other clients can not reach your server, if they try to address you via ipv4, since it is not converted back to ipv6 the other way around
- So, this is only a one way conversion (the tunneling)
- And since there are still many devices, which do only use ipv4, this is a problem for us, more about this later: [#Dynamic Global ipv6 Address]({{< ref "#dynamic-global-ipv6-address" >}})

### IP Addresses

Some infos about IP addresses, mostly for myself...
- Ipv4 is 32 bit (4 * 8bits); ipv6 is 128 bit (8 * 16bits); ipv6 has 64 bit prefix
- local link ipv6 is not the same as the local network ipv4 addresses

Ip4 local addresses typical range:
```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

ipv6 range local normal (Unique local addresses), not usable:
```
FC00::/7
```

ipv6 link local:
```
fe80:/10 #but 64 bit prefix, since rest 0
```

ipv6 global, for us:
```
2a0:
```


## Server Global Accessible
You only need to read this section if you want others to be able to access your server. If you are the only one, go for a VPN connection, like the one with tailscale as explained here: [Raspberry Pi#Tailscale VPN]({{< ref "Raspberry Pi#tailscale-vpn" >}})

### Router Port Forwarding
In the settings of your router, you have to configure, which traffic (ports) shall be allowed to which devices
- For nomal http and https traffic, you have to open port 80 (http) and 443 (hhtps) for your server
- Note that in some routers it is not possible to open a port for a specific device, but only rather for a specific ip address
      - If that is the case you have to put in the global ip address of your server
      - Note: This can be problematic if your global IP address is changing; In that case you have to manually open the port for the new IP address again, every time it is changing :/
      - The normal vodafone router, can only open ports for specific IP addresses :/
      - A fritzbox can open ports for specific devices :)
- Try to expose as little ports as possible
- General list of usable ports [wiki port list](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers)

Test your settings:
- Get your global ipv6 address of your server
	- `ip addr`
- Connect with a device to your mobile hotspot or so (you have to somehow not in your local network)
- Try to ping the ipv6 address via an open port (80 or so)
	- `nmap -p 80 -6 2a02:908:...:...`
- The result should show that the host is up and the port is open

### Dynamic Global ipv6 Address
As described above, we often only have a dynamically changing ipv6 address (due to dual stack light) and somehow have to make our server available
- First step is to cope with the problem of the dynamically changing IP address
- Second, solve the problem with the dual stack light (ipv6 only)

#### DynDNS and Strato
To solve the problem with the dynamic IP address, we can use a domain, which will automatically update the pointer to our server; So it will update the IP address it is pointing to every time our IP address is changing.

You can e.g. buy a domain at strato.de and then you can use strato to keep track of your dynamically changing ip address:
- Buy a domain and in settings of your domain enable dynamic dns
	- Optionally: to make the process faster (dyndns update might take some time), before enabling dyndns set the AAAA record manually
	    - get the global ipv6 address of your raspi `sudo ip addr`
	    - enter it manually as the AAAA record
	    - now enable dyndns - the record will be overwritten eventually
- Install a dynamic dns client on your server to forward changing IP addresses to strato.de
	- `sudo apt install ddclient`
	- You can ignore the pop up during the installation and just skip or put in some dummy infos, since you will delete the config file directly
	- `sudo nano /etc/ddclient.conf`
```
mail=mail address
protocol=dyndns2
usev6=if, if=eth0
ssl=yes
server=dyndns.strato.com/nic/update

# Your domain
login=your.domain.name
password='yourpassword'
your.domain.name

# Add another domain if you want to update multiple domains
#login=another.domain.name
#password='yourpassword'
#another.domain.name
```
- .
	- update ddclient config
	- `sudo ddclient --force`
	- Sometimes it is also usefull to fully restart ddclient and reset cache
    - `sudo systemctl stop ddclient.service && rm -rf /var/cache/ddclient/ddclient.cache && systemctl restart ddclient.service`
    - And check status via this command: `sudo systemctl status ddclient.service`
- ddclient will also try to notify you via mail if the ip address is changed. However, for this to work, a system wide mail service must be installed, as described under [Raspberry Pi#Mail server]({{< ref "Raspberry Pi#mail-server" >}})
- Maybe the update is not directly visible on strato since it takes some time until they process changes

After this step you already have a domain which will always point to your ipv6 address :)

Test your settings:
- Try to ping your domain via ipv6
	- `ping6 mydomain.de`

#### Feste-IP
We can use fest-ip.net to convert ipv4 queries into ipv6 queries. They offer this service, but you have to pay for it.

If you also want to be reachable via ip4 you can:
- Buy a dedicated portmapper (an own ipv4 address) under credits and addons at feste-ip.net
- Under universal portmapper, create a new mapping
	- map your newly bought own ipv4 address to your domain (e.g. bergrunde.net) and map 80 to 80 and 443 to 443 and so on, depending on the ports you are using
  - Now you have to let your domain point to the ipv4 address you just bought
	- go to strato again and deactivate dyndns
	- manually open the A record and enter your ipv4 address
	- Optionally: to the same again for AAAA record as described above to not have to wait for the update of the ddclient
	- Now enable dyndns again
	- Only the ipv6 address gets overwritten again at some time and the ipv4 will be always the manual set one
- Done: You can now be reached via ipv4 and ipv6
	- ipv4 is redirected to feste-ip.net where it will redirect the ipv4 traffic to ipv6 and forward it to your domain which will always point to your real ipv6 address, since it is always up2date thanks to dyndns

Test your settings:
- Try to ping your domain via ipv4
	- `ping -4 mydomain.de`

### HTTPS / SSL enryption
[HTTPS](https://en.wikipedia.org/wiki/HTTPS) is used to encrypt http internet traffic via ssl
- Since we want to have a secure connection for our servers, we also want to use https instead of unencrypted http traffic
- Our server and the client have to perform a handshake to exchange the public key, in order to sucessfull encrypt via ssl
	- This goes in hand with having a website certificate, which kind of also represents the public key together with other information about your server
- To make this work, we have to possibilities
	- We can create a self signed certificate, which allows us to create a safe https communication
		- However, this certificate is not signed by an authority and consequently users will get a pop up, that your website is not safe, since they can not verify if the domain they are accessing is really the website they wanted to access
		- For you this is no problem, because you know that this is your domain and the connection is still safe.
	- We can create a certificate, which is signed by an authority
		- This way, your domain will be verified, that you have full control over it and that you are really the owner of this domain
		- After that every one who is accessing your website will see that it is secure to do so
		- It can be costly to get an certificate from an authority. However, there is [letsencrypt](https://letsencrypt.org/) which is a non-profit organization, which is doing this for free, since they have the mission to make the internet a more secure space
		- I can highly **recommend** to use **letsencrypt** and once you know about this organization, you will realize that many websites you might visit often are actually using letsencrypt. Here are two ways to use letsencrypt
			- You can follow their [getting started guide](https://letsencrypt.org/getting-started/) and do everything manually
			- Or do it as I do it and use the webserver caddy, to do everything automatically for you, which I highly recommend
				- Read more about this, in the section about the [caddy webserver]({{< ref "Docker and Server Apps#caddy" >}})

## DNS Global
- DNS servers are used to get the IP address, which a domain is pointing to
- Normally, your router is automatically using the DNS server of your internet provider
- DNS queries are not encrypted, so normally everyone can see which sites you are requesting
	- Once a connection to a website is established, the detailed traffic is not visible anymore since it is https encrypted
- First step to make your traffic a little bit more secure, is to use a different DNS server, like cloudflared, which will most likely not use your internet requests to profile you
	- Though, the requests are still not encrypted and might be accessed in the middle
- Second step is to encrypt your DNS queries, which is supported by cloudflared, such that no one can know which sites you are requesting
	- During this step it is necessary to create an own local DNS server, which will handle the encryption and forward the traffic to cloudflared
	- A good setup is to use this in combination with a pi-hole DNS server, see here for more information on how to set it up on your RasPi via docker: [Docker and Server Apps#Pi-hole and Cloudflared]({{< ref "Docker and Server Apps#pi-hole-and-cloudflared" >}})

### Change DNS
- Here a guide on how to change the used DNS server on your Raspberry: [Raspberry Pi#DNS]({{< ref "Raspberry Pi#dns" >}})
- And here for Ubuntu: [#DNS Ubuntu]({{< ref "#dns-ubuntu" >}})
- I personlay have changed the DNS server within my router settings (**recommended**), this way every device that is connected to my local network automatically uses my local Pi-hole as a DNS server, since a device in the local network is asking the router by default to tell it the DNS server it shall use
	- I additionally can also use my Pi-hole from anywhere else, since I have all my devices always connected to my Pi-Hole via tailscale DNS ([Raspberry Pi#Tailscale DNS]({{< ref "Raspberry Pi#tailscale-dns" >}}))

Here some infos for testing:
- DNS records can be queried via ipv4 or ipv6 (connection to the DNS server), test with `dig -4 example.com` and `dig -6 example.com`.
	- Note: How to communicate with the domain (v4 vs v6) is described by the DNS record "A" vs "AAAA". To test ipv6 use `ping6 example.com`. This will query the ipv6 address from the DNS sever, doesn't matter if communication is via ipv4 or ipv6 with DNS sever, or use `dig example.com aaaa`.
- To change your DSN sever to a DNS server in your local network, it is enough to communicate with it via ipv4. No need to setup ipv6 communication with it, you can query ipv6 addresses / AAAA records via ipv4 in this case, anyway

## Linux Network
### Restart Network
```
sudo service network-manager restart
sudo systemctl restart NetworkManager.service
```

### Network Commands
Ping all devices and see who is up:
```
nmap 192.168.0.0/24
```

Other Commands:
```
hostname -I
ip addr
ip link set wlan0 down
ip link set wlan0 up
```

### DNS Ubuntu

Note that on Ubuntu, `systemd-resolved` is as internal DNS server (proxy), listening on `127.0.0.53` for DNS queries to forward to configured DNS sever:
- configuration via GUI is possible for the DNS server, you can also disable ipv6 and only configure ipv4 DNS server; just open your current Network connection via the GUI and go to the DNS tab
- you can check the currently used one via command line, but do not change here: `sudo nano /run/systemd/resolve/resolv.conf`
	- see `man 5 resolved.conf`
	- local overwrites in `sudo nano /etc/systemd/resolved.conf`

### mDNS local dns
https://stevessmarthomeguide.com/multicast-dns/
https://github.com/george-hawkins/avahi-aliases-notes
avahi needed as program
