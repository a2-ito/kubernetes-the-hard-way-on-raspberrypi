# 13-operations for etcd

## recovery existing failure node  

### check health 
```
for instance in 1 2 3; do
  ssh rasp-etcd-${instance} "\
  sudo ETCDCTL_API=3 etcdctl endpoint health \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/etcd/ca.pem \
    --cert=/etc/etcd/kubernetes.pem \
    --key=/etc/etcd/kubernetes-key.pem
  "
done
```

```
for instance in 1 2 3; do
  ssh rasp-etcd-${instance} "\
  sudo ETCDCTL_API=3 etcdctl member list \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/etcd/ca.pem \
    --cert=/etc/etcd/kubernetes.pem \
    --key=/etc/etcd/kubernetes-key.pem
  "
done
```

### remove member 
You just need to remove and add operation on only one node.

```
sudo ETCDCTL_API=3 etcdctl member remove c43eaf24a8732c53 \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

### add member
```
sudo ETCDCTL_API=3 etcdctl member add rasp-etcd-3 \
  --peer-urls=https://192.168.11.13:2380 \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

### edit etcd service

```
pi@rasp-etcd-3:~ $ cat /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Environment=ETCD_UNSUPPORTED_ARCH=arm
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name rasp-etcd-3 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.11.13:2380 \
  --listen-peer-urls https://192.168.11.13:2380 \
  --listen-client-urls https://192.168.11.13:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.11.13:2379 \
  --initial-cluster-token etcd-cluster \
  --initial-cluster rasp-etcd-1=https://192.168.11.11:2380,rasp-etcd-2=https://192.168.11.12:2380,rasp-etcd-3=https://192.168.11.13:2380 \
  --initial-cluster-state existing \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### start etcd



