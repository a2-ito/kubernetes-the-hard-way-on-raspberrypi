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
sudo rm /etc/localtime
sudo rm /etc/timezone
sudo sh -c 'echo Asia/Tokyo > /etc/timezone'
sudo dpkg-reconfigure -f noninteractive tzdata
```

```
sudo vi /etc/dhcpcd.conf
sudo vi /etc/hostname
sudo vi /etc/hosts
```

```
sudo apt-get update
```

## Add Public key to connect via SSH without password
```
mkdir .ssh
vi .ssh/authorized_keys
```

## Change Default Password
```
passwd
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

