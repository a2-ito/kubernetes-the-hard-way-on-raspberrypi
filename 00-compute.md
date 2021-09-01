# 00-compute

## list of k8s nodes

| hostname         | IP            |
|:-----------------|:--------------|
| rasp-etcd-1      | 192.168.11.11 |
| rasp-etcd-2      | 192.168.11.12 |
| rasp-etcd-3      | 192.168.11.13 |
| rasp-k8s-master-1| 192.168.11.14 |
| rasp-worker-1    | 192.168.11.21 |
| rasp-worker-2    | 192.168.11.22 |
| rasp-worker-3    | 192.168.11.23 |
| rasp-worker-4    | 192.168.11.24 |

```
for i in `seq 1 254`
do
  echo 192.168.11.${i}
  ping -c 1 -t 100 192.168.11.${i}
done
```


```
sudo rm /etc/localtime
sudo rm /etc/timezone
sudo sh -c 'echo Asia/Tokyo > /etc/timezone'
sudo dpkg-reconfigure -f noninteractive tzdata
```

### Case 1. Raspbian

```
sudo vi /etc/dhcpcd.conf
```

#### ex.
```
interface eth0
static ip_address=192.168.11.11/24
static routers=192.168.11.1
static domain_name_servers=8.8.8.8
```
### Case 2. Ubuntu

```
sudo vi /etc/netplan/99_config.yaml
```

```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false
      addresses: [192.168.11.21/24]
      gateway4: 192.168.11.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

### Edit Hostname
```
sudo vi /etc/hostname
sudo vi /etc/hosts
```

```
sudo apt update
```

## Add Public key to connect via SSH without password

### only for Ubuntu
```
sudo adduser pi
sudo sh -c "echo 'pi ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers.d/90-cloud-init-users"
sudo cat /etc/sudoers.d/90-cloud-init-users
```

```
sudo su - pi
mkdir .ssh
vi .ssh/authorized_keys
```

## Change Default Password
```
passwd
```

## SSH Tuning
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo cp -p /etc/ssh/sshd_config /tmp/sshd_config.bk
    sudo sed -i 's/#UseDNS no/UseDNS no/' /etc/ssh/sshd_config
    sudo sed -i 's/#AddressFamily any/AddressFamily inet/' /etc/ssh/sshd_config
    sudo sed -i 's/UsePAM yes/UsePAM no/' /etc/ssh/sshd_config
    diff /etc/ssh/sshd_config /tmp/sshd_config.bk
    sudo systemctl reload sshd
  "
done
```

```
for instance in 1 2 3; do 
  ssh rasp-worker-${instance} "\
    hostname
    hostname -i
    id -a
  "
done
```

## Edit hosts
```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
  sudo sh -c \"echo 192.168.11.21 rasp-worker-1 >> /etc/hosts\"
  sudo sh -c \"echo 192.168.11.22 rasp-worker-2 >> /etc/hosts\"
  sudo sh -c \"echo 192.168.11.23 rasp-worker-3 >> /etc/hosts\"
  "
done
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    cat /etc/hosts
  "
done
```

## Overscan configuration

overscan configuration for micro-monitor

### Raspbian
```
sudo vi /boot/config.txt
```

```
overscan_left=-32
overscan_right=-32
overscan_top=-32
overscan_bottom=96
```

