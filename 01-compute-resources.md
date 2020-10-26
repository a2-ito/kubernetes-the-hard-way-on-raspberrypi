# Compute Resources

## Install Raspbian Lite





## IP Configuration
- Master Nodes
  - 192.168.11.101
  - 192.168.11.102
  - 192.168.11.103

/etc/dhcpcd.conf
```
interface eth0
static ip_address=192.168.11.3/24
static routers=192.168.11.1
static domain_name_servers=8.8.8.8
```

/etc/hostname
```
rasp-master-1
rasp-master-2
rasp-master-3
```
```
$ sudo rm /etc/localtime
$ sudo rm /etc/timezone
$ sudo sh -c 'echo Australia/Sydney > /etc/timezone'
$ sudo dpkg-reconfigure -f noninteractive tzdata
```

## aaa
```
sudo apt-get update
sudo apt-get upgrade
sudo reboot
```

## SSH config
```
$ ssh-keygen -t rsa -b 4096 -C "hi.mound@gmail.com"
```
cat ~/.ssh/rasp_rsa.pub

~/.ssh/authorized_keys

vi /etc/sshd/config
```
Host rasp-master-1
    Hostname 192.168.11.11
    User pi
    Port 22
    IdentityFile ~/.ssh/keys/rasp_rsa
Host rasp-master-2
    Hostname 192.168.11.11
    User pi
    Port 22
    IdentityFile ~/.ssh/keys/rasp_rsa
Host rasp-master-3
    Hostname 192.168.11.13
    User pi
    Port 22
    IdentityFile ~/.ssh/keys/rasp_rsa
```




