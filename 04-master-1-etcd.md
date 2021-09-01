# 00-kubernetes-controller

## Install etcd binaries
### Install etcd binaries
```
for instance in 1 2 3; do
  ssh rasp-etcd-${instance} "\
    sudo mkdir -p /etc/etcd /var/lib/etcd
    sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
    wget https://raw.githubusercontent.com/robertojrojas/kubernetes-the-hard-way-raspberry-pi/master/etcd/etcd-3.1.5-arm.tar.gz
    tar xzf etcd-3.1.5-arm.tar.gz
    sudo mv etcd-3.1.5-arm/etcd* /usr/local/bin/
    "
done
```
### Option - Install latest version
#### Build


#### Deploy
```
for instance in 1 2 3; do
  scp ./build/* rasp-etcd-${instance}:~
  ssh rasp-etcd-${instance} "\
    sudo mkdir -p /etc/etcd /var/lib/etcd
    sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
    sudo mv etcd* /usr/local/bin/
    sudo chmod 775 /usr/local/bin/etcd*
    "
done
```
#### Verification
```
for instance in 1 2 3; do
  ssh rasp-etcd-${instance} "\
    ETCD_UNSUPPORTED_ARCH=arm /usr/local/bin/etcd --version
    "
done
```

## Configure the ectd Server
### Craete the systemd unit file
```
for instance in 1 2 3; do
cat > etcd.service <<"EOF"
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Environment=ETCD_UNSUPPORTED_ARCH=arm
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name ETCD_NAME \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://INTERNAL_IP:2380 \
  --listen-peer-urls https://INTERNAL_IP:2380 \
  --listen-client-urls https://INTERNAL_IP:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://INTERNAL_IP:2379 \
  --initial-cluster-token etcd-cluster \
  --initial-cluster rasp-etcd-1=https://192.168.11.11:2380,rasp-etcd-2=https://192.168.11.12:2380,rasp-etcd-3=https://192.168.11.13:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
  export INTERNAL_IP=192.168.11.1${instance}
  export ETCD_NAME=rasp-etcd-${instance}
  sed -i s/INTERNAL_IP/${INTERNAL_IP}/g etcd.service
  sed -i s/ETCD_NAME/${ETCD_NAME}/g etcd.service
  scp etcd.service rasp-etcd-${instance}:~
  rm etcd.service
  ssh rasp-etcd-${instance} "sudo mv etcd.service /etc/systemd/system/"
done
```

## Enable and Start etcd
```
for instance in 1 2 3; do
  ssh rasp-etcd-${instance} "\
    sudo systemctl daemon-reload
    sudo systemctl enable etcd
    sudo systemctl start etcd &
  "
done
```

## Verification
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
