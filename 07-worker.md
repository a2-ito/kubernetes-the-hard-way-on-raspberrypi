# Kubernetes the hard way

## Enable Memory Subsysem of cgroup
```
vi /boot/firmware/cmdline.txt
```

You need to add a following setting.
```
cgroup_memory=1 cgroup_enable=memory
```

## Copy Kubeconfigs
```
for instance in 1 2 3; do
  scp kube-proxy.kubeconfig rasp-worker-${instance}.kubeconfig rasp-worker-${instance}:~/
done
```
## Copy Keys
```
for instance in 1 2 3; do
  scp ./keys/ca.pem \
      ./keys/rasp-worker-${instance}.pem \
      ./keys/rasp-worker-${instance}-key.pem \
      rasp-worker-${instance}:~/
done
```

## Provisioning a Kubernetes Worker Node
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
  	sudo apt-get update
  	sudo apt-get -y install socat conntrack ipset
  "
done
```
## Disable Swap
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo swapoff -a
    sudo swapon --show
  "
done
```

## OPTION: Disable Swap permanently
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo systemctl disable dphys-swapfile
  "
done
```

## Download and Install Worker Binaries
```
K8S_VER=v1.18.18
K8S_ARCH=arm64
mkdir bin-for-workers
wget -q --show-progress --https-only --timestamping -P ./bin-for-workers \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.18.0/crictl-v1.18.0-linux-${K8S_ARCH}.tar.gz \
  https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-${K8S_ARCH}-v0.8.6.tgz \
  https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kubelet
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo apt -y install containerd runc
  "
  scp ./bin-for-workers/* rasp-worker-${instance}:~
done
```

### Create the installation directories
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo mkdir -p \
      /etc/cni/net.d \
      /opt/cni/bin \
      /var/lib/kubelet \
      /var/lib/kube-proxy \
      /var/lib/kubernetes \
      /var/run/kubernetes \
      /var/log/kubernetes \
      /etc/containerd
    "
done
```
### Install the worker binaries
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo tar -xvf cni-plugins-linux-arm64-v0.8.6.tgz -C /opt/cni/bin/
    tar xzf crictl-v1.18.0-linux-arm64.tar.gz
    chmod +x crictl kubectl kube-proxy kubelet
    sudo mv crictl kubectl kube-proxy kubelet /usr/local/bin/
  "
done
```

```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    /opt/cni/bin/loopback version
    crictl --version
    kubelet --version
    kube-proxy --version
  "
done
```

### DNS
<details>

```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo mkdir -p /run/systemd/resolve/
    sudo ln -s /etc/resolv.conf /run/systemd/resolve/resolv.conf
  "
done
```

</details>

## Configure the Kubelet

```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo mv rasp-worker-${instance}-key.pem rasp-worker-${instance}.pem /var/lib/kubelet/
    sudo mv rasp-worker-${instance}.kubeconfig /var/lib/kubelet/kubeconfig
    sudo mv ca.pem /var/lib/kubernetes/
  "
done
```

### Option 1. Docker
<details>

#### CNI

```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo mkdir -p /opt/cni
    wget https://raw.githubusercontent.com/robertojrojas/kubernetes-the-hard-way-raspberry-pi/master/cni/cni.tar.gz
    sudo tar -xvf cni.tar.gz -C /opt/cni
  "
done
```

### Create the ```kubelet-config.yaml``` configurations file
```
for instance in 1 2 3; do
POD_CIDR=10.200.${instance}.0/24
cat <<EOF | tee kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
#resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/rasp-worker-${instance}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/rasp-worker-${instance}-key.pem"
EOF
  scp kubelet-config.yaml rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv kubelet-config.yaml /var/lib/kubelet/
  "
rm kubelet-config.yaml
done
```

#### Configure kubelet with Docker
```
cat > kubelet.service <<"EOF"
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --cloud-provider= \
  --cluster-dns=10.32.0.10 \
  --cluster-domain=cluster.local \
  --container-runtime=docker \
  --docker=unix:///var/run/docker.sock \
  --network-plugin=kubenet \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --serialize-image-pulls=false \
  --v=2
  
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
for instance in 1 2 3; do
  scp kubelet.service rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv kubelet.service /etc/systemd/system/
  "
done
rm kubelet.service
```

</details>

### Option 2. CNI

<details>

### Retrieve the Pod CIDR range for the current compute instance

```
cat << EOF | tee config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/sbin/runc"
      runtime_root = ""
EOF
for instance in 1 2 3; do
  scp config.toml rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv config.toml /etc/containerd/
  "
done
```

```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo systemctl start containerd
    sudo systemctl enable containerd
  "
done
```

### Install CNI
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo mkdir -p \
      /etc/cni/net.d \
      /opt/cni/bin
    wget -q --show-progress --https-only --timestamping -P ./bin-for-workers \
      https://github.com/containernetworking/plugins/releases/download/v0.8.7/cni-plugins-linux-arm-v0.8.7.tgz 
    sudo tar -xvf cni-plugins-linux-arm-v0.8.7.tgz -C /opt/cni/bin/
    /opt/cni/bin/loopback version
  "
done
```

### Create the ```bridge``` network configuration file

```
for instance in 1 2 3; do
POD_CIDR=10.200.${instance}.0/24
cat <<EOF | tee 10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
  scp 10-bridge.conf rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv 10-bridge.conf /etc/cni/net.d/
  "
  rm 10-bridge.conf
done
```

### Create the ```loopback``` network configuration file
```
cat <<EOF | tee 99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF
for instance in 1 2 3; do
  scp 99-loopback.conf rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv 99-loopback.conf /etc/cni/net.d/
  "
done
rm 99-loopback.conf
```

### Create the ```kubelet-config.yaml``` configurations file
```
for instance in 1 2 3; do
POD_CIDR=10.200.${instance}.0/24
cat <<EOF | tee kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
#resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/rasp-worker-${instance}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/rasp-worker-${instance}-key.pem"
EOF
  scp kubelet-config.yaml rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv kubelet-config.yaml /var/lib/kubelet/
  "
rm kubelet-config.yaml
done
```

#### Configure the Kubelet with CNI
```
cat > kubelet.service <<"EOF"
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
  --image-pull-progress-deadline=2m \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --network-plugin=cni \
  --register-node=true \
  --authentication-token-webhook=true \
  --authorization-mode=Webhook \
  --v=2 \
  --log-dir=/var/log/kubernetes/ \
  --log-file=/var/log/kubernetes/kubelet.log \
  --logtostderr=false
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
for instance in 1 2 3; do
  scp kubelet.service rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv kubelet.service /etc/systemd/system/
  "
done
rm kubelet.service
```

</details>

### Create the ```kubelet.service``` systemd unit file

#### Option 2. docker
#### Option 3. docker-2
```
cat > kubelet.service <<"EOF"
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --allow-privileged=true \
  --api-servers=https://10.0.1.94:6443,https://10.0.1.95:6443,https://10.0.1.96:6443 \
  --cloud-provider= \
  --cluster-dns=10.32.0.10 \
  --cluster-domain=cluster.local \
  --configure-cbr0=true \
  --container-runtime=docker \
  --docker=unix:///var/run/docker.sock \
  --network-plugin=kubenet \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --reconcile-cidr=true \
  --serialize-image-pulls=false \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2
  
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
for instance in 1 2 3; do
  scp kubelet.service rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv kubelet.service /etc/systemd/system/
  "
done
rm kubelet.service
```


### Start the Worker Services - kubelet
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo systemctl daemon-reload
    sudo systemctl enable kubelet
    sudo systemctl start kubelet
  "
done
```

```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo systemctl status kubelet -l
  "
done
```

If kubelet doesn't work, You need to add can enable this into ```/boot/cmdline.txt```.
```
cgroup_enable=memory
```

## Configure the Kubernetes Proxy
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
  "
done
```
### Create the ```kube-proxy-config.yaml``` configuration file
```
cat <<EOF | tee kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
iptables:
  masqueradeAll: true
EOF
for instance in 1 2 3; do
  scp kube-proxy-config.yaml rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv kube-proxy-config.yaml /var/lib/kube-proxy
  "
done
rm kube-proxy-config.yaml
```
### Create the ```kube-proxy.service``` systemd unit file
```
cat <<EOF | tee kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml \
  --log-dir=/var/log/kubernetes/ \
  --log-file=/var/log/kubernetes/kube-proxy.log \
  --logtostderr=false
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
for instance in 1 2 3; do
  scp kube-proxy.service rasp-worker-${instance}:~
  ssh rasp-worker-${instance} "\
    sudo mv kube-proxy.service /etc/systemd/system/
  "
done
rm kube-proxy.service
```

### Start the Worker Services
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo systemctl daemon-reload
    sudo systemctl enable kube-proxy
    sudo systemctl start kube-proxy
  "
done
```
```
for instance in 1 2 3; do
  ssh rasp-worker-${instance} "\
    sudo systemctl status kube-proxy
  "
done
```

## Verification
```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    kubectl get nodes --kubeconfig admin.kubeconfig
  "
done
```
```
kubectl run --kubeconfig admin.kubeconfig \
  --generator=run-pod/v1 busybox --image=busybox:1.28 --command -- sleep 3600
kubectl exec --kubeconfig admin.kubeconfig -it busybox -- sh
```

```
kubectl get pod --kubeconfig admin.kubeconfig
kubectl delete pod busybox --kubeconfig admin.kubeconfig
```

```
kubectl run --image=busybox:1.28 --rm --restart=Never -i testpod \
  --overrides='{"spec": { "nodeSelector": {"kubernetes.io/hostname": "rasp-worker-1"}}}' \
  -- nslookup kubernetes
kubectl run --image=busybox:1.28 --rm --restart=Never -i testpod \
  --overrides='{"spec": { "nodeSelector": {"kubernetes.io/hostname": "rasp-worker-3"}}}' \
  -- nslookup kubernetes
kubectl run --image=busybox:1.28 --rm --restart=Never -i testpod \
  --overrides='{"spec": { "nodeSelector": {"kubernetes.io/hostname": "rasp-worker-3"}}}' \
  -- nslookup kubernetes
```


