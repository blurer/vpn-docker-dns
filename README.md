# vpn-docker-dns
Setup host with Docker, Wireguard, and PiHole

Spin up VM on provider of choice. 

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

## Setup User ##

useradd bl
usermod -aG sudo bl
mkhomedir_helper bl
mkdir /home/bl/.ssh/
wget https://raw.githubusercontent.com/blurer/myBS/master/authorized_keys 
mv authorized_keys /home/bl/.ssh/authorized_keys
chmod 600 /home/bl/.ssh/authorized_keys
chown -R bl:bl /home/bl/.ssh/
passwd bl {password}

## Login as user ##

sudo su bl
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

sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

cd $HOME/
wget https://raw.githubusercontent.com/blurer/myBS/master/zshrc
mv zshrc .zshrc

sudo systemctl enable fail2ban

source .zshrc

sudo ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
## Docker ##

curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

sudo usermod -aG docker bl

## IF RUNNING DNS ##
Fix port 53 since systemd-resolved blocks PiHole/AdguardHome.

sudo sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/g' /etc/systemd/resolved.conf 	 
sudo sed -r -i.orig 's/#DNS=/DNS=1.1.1.1/g' /etc/systemd/resolved.conf 	 
sudo sed -r -i.orig 's/#FallbackDNS=/DNS=8.8.8.8/g' /etc/systemd/resolved.conf 	
sudo sh -c 'rm /etc/resolv.conf && ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf'
sudo systemctl restart systemd-resolved


#REBOOT#

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

To suppress sudo warnings: "sudo: unable to resolve host bl-jp: Name or service not known"
edit /etc/hosts with your new hostname.


## VPN ##
