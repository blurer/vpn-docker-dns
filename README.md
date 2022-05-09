# vpn-docker-dns

I need to re-write https://github.com/blurer/Homelab-Setup to be used based (other than mine) within the docker files. In the meantime, this is base setup for a new host with Docker, Wireguard, and PiHole

## Start
Spin up VM on provider of choice. If using Portainer and Proxy, go with at least 1gb, otherwise 512mb will work for just PiHole, Doker,and VPN.

Open the following ports:

| Port | Type | Use | IPs
|---|---|---|---|
| 53 | TCP | DNS | 0.0.0.0/0, ::0/0
| 53 | UDP | DNS | 0.0.0.0/0, ::0/0
| 51820 | UDP | Wireguard |  0.0.0.0/0, ::0/0
| 1194 | UDP | OpenVPN |  0.0.0.0/0, ::0/0
| 5201 | UDP | iperf |  0.0.0.0/0, ::0/0
| 5201 | TCP | iperf |  0.0.0.0/0, ::0/0
| 80 | TCP | HTTP |  0.0.0.0/0, ::0/0
| 443 | TCP | HTTPS |  0.0.0.0/0, ::0/0
| 81 | TCP | Proxy Mgmt | Home WAN IP 

Make sure you set static IP (e.g. EC2/Lightsail) and enable IPv6.

## Setup User
```
useradd bl
usermod -aG sudo bl
mkhomedir_helper bl
mkdir /home/bl/.ssh/
wget https://raw.githubusercontent.com/blurer/myBS/master/authorized_keys 
mv authorized_keys /home/bl/.ssh/authorized_keys
chmod 600 /home/bl/.ssh/authorized_keys
chown -R bl:bl /home/bl/.ssh/
passwd bl {password}
```
## Login as user 
```
sudo su bl
sudo ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
mkdir /home/bl/docker/
sudo apt update -y
sudo apt upgrade -y
sudo apt install -y \
	vim \
	git \
	curl \
	htop \
	ansible \
	unzip \
	zsh \
	python3-pip \
	python3-setuptools \
	fail2ban 
sudo systemctl enable fail2ban

vim /etc/hostname -> {new hostname}
vim /etc/hosts -> add 127.0.0.1 {new-hostname}
```

Setup ohmyzsh: 
``sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"``
```
cd $HOME/
wget https://raw.githubusercontent.com/blurer/myBS/master/zshrc
mv zshrc .zshrc
source .zshrc
```


## Docker
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker bl
sudo systemctl enable docker
sudo apt install docker-compose -y
```

## Fix Systemd-resolved
Fix port 53 since systemd-resolved blocks PiHole/AdguardHome. If dont do this, PiHole will be unable to start as port 53 is already in use.

sudo sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/g' /etc/systemd/resolved.conf 	 
sudo sed -r -i.orig 's/#DNS=/DNS=1.1.1.1/g' /etc/systemd/resolved.conf 	 
sudo sed -r -i.orig 's/#FallbackDNS=/DNS=8.8.8.8/g' /etc/systemd/resolved.conf 	
sudo sh -c 'rm /etc/resolv.conf && ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf'
sudo systemctl restart systemd-resolved

## Reboot

CHECK TO MAKE SURE DOCKER AND DNS ARE FUNCTIONAL:
```
# bl @ bl-jp in ~ [7:41:08] C:1
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

# bl @ bl-jp in ~ [7:41:09]
$ nslookup google.com
Server:         1.1.1.1
Address:        1.1.1.1#53

Non-authoritative answer:
Name:   google.com
Address: 172.217.161.206
Name:   google.com
Address: 2404:6800:4004:827::200e
```

## VPN 
``curl -L https://install.pivpn.io | bash``

Run through setup - I keep default port, and use local resolver (2nd from bottom option) for DNS. 
To create new users, ``pivpn -a``, to get the QR code for mobile ``pivpn -qr``, finally to see stats, ``pivpn -c``

Example
```
$ pivpn -c
::: Connected Clients List :::
Name           Remote IP       Virtual IP                                Bytes Received      Bytes Sent      Last Seen
home-wan       (none)          10.209.73.2,fd11:5ee:bad:c0de::2/128      0B                  0B              (not yet)
bl-iphone      x.x.x.x:x       10.209.73.3,fd11:5ee:bad:c0de::3/128      610MiB              2.5GiB          May 09 2022 - 12:44:35
bl-lt          x.x.x.x:x       10.209.73.4,fd11:5ee:bad:c0de::4/128      298MiB              1.3GiB          May 09 2022 - 17:51:37
bl-ipad        x.x.x.x:x       10.209.73.5,fd11:5ee:bad:c0de::5/128      299MiB              9.9GiB          May 09 2022 - 17:49:47
```

## DNS

Grab my config here: https://raw.githubusercontent.com/blurer/Homelab-Setup/main/files/dns.yml

Note: Be sure to change your timezone, password, and ports as applicable. 

Found these block a majority of ads, trackers, etc - without breaking the internet. 
- Blocklist: https://raw.githubusercontent.com/blurer/Homelab-Setup/main/pihole/blocks.txt
- Whitelist: https://raw.githubusercontent.com/blurer/Homelab-Setup/main/pihole/whitelist.txt

## Portainer (Optional)
Great tool to manage the Docker containers via a web interface. 

``docker run -d -p 8000:8000 -p 9443:9443 -p 9000:9000 --name portainer    --restart=always    -v /var/run/docker.sock:/var/run/docker.sock    -v $HOME/docker/portainer/:/data  portainer/portainer-ce:latest``

## Nginx Proxy Manager

```
mkdir $HOME/docker/proxy/
cd $HOME/docker/proxy
wget https://raw.githubusercontent.com/blurer/Homelab-Setup/main/files/proxy.yml
docker-compose -f $HOME/docker/proxy/docker-compose.yml up -d
```
