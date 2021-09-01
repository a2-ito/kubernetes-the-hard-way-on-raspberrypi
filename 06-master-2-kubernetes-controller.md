# 04-master-2-kubernetes-control-plane

## Copy Keys
```
for instance in 1; do
  scp ./keys/ca.pem \
      ./keys/ca-key.pem \
      ./keys/kubernetes-key.pem \
	  ./keys/kubernetes.pem \
	  ./keys/service-account-key.pem \
	  ./keys/service-account.pem \
      rasp-k8s-master-${instance}:~/
done
```

## The Encryption Config File
```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
for instance in 1; do
  scp encryption-config.yaml rasp-k8s-master-${instance}:~
done
```

## Download and Install the Kubernetes controller
```
K8S_VER=v1.18.18
K8S_ARCH=arm
wget \
  "https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kubectl"
for instance in 1; do
  scp kube-apiserver kube-controller-manager kube-scheduler kubectl rasp-k8s-master-${instance}:~
done
```
## Install the Kubernetes binaries
```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    sudo mkdir -p /etc/kubernetes/config
    chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
    sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin
  "
done
```
```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    kube-apiserver --version
    kube-controller-manager --version
    kube-scheduler --version
  "
done
```
## Configure the Kubernetes API Server
```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    sudo mkdir -p /var/lib/kubernetes/
    sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
      service-account-key.pem service-account.pem \
      encryption-config.yaml /var/lib/kubernetes
  "
done
```

### Create the ```kube-apiserver.service``` systemd unit file
```
for instance in 1; do
  INTERNAL_IP=192.168.11.14
  cat << EOF | tee kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.11.11:2379,https://192.168.11.12:2379,https://192.168.11.13:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
  scp kube-apiserver.service rasp-k8s-master-${instance}:~
  rm kube-apiserver.service
  ssh rasp-k8s-master-${instance} "sudo mv kube-apiserver.service /etc/systemd/system/"
done
```

```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    sudo systemctl daemon-reload
    sudo systemctl enable kube-apiserver
    sudo systemctl start kube-apiserver &
  "
done
```

```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    sudo systemctl status kube-apiserver -l
  "
done
```

### Configure the Kubernetes Controller Manager
```
for instance in 1; do
 ssh rasp-k8s-master-${instance} "sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/"
done
```
#### Create the ```kube-controller-manager.service``` systemd unit file
```
for instance in 1; do
  INTERNAL_IP=192.168.11.14
  cat << EOF | tee kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
  scp kube-controller-manager.service rasp-k8s-master-${instance}:~
  rm kube-controller-manager.service
  ssh rasp-k8s-master-${instance} "\
     sudo mv kube-controller-manager.service /etc/systemd/system/
  "
done
```

```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    kube-controller-manager --version
  "
done
```

```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    sudo systemctl daemon-reload
    sudo systemctl enable kube-controller-manager
    sudo systemctl start kube-controller-manager &
  "
done
```

#### Verification
```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    sudo systemctl status kube-controller-manager -l
  "
done
```
```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    kubectl get componentstatuses
  "
done
```

### Configure the Kubernetes Scheduler

#### Create the ```kube-scheduler.yaml``` configuration file
```
cat <<EOF | tee kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
cat <<EOF | tee kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
for instance in 1; do
  scp kube-scheduler.yaml kube-scheduler.service kube-scheduler.kubeconfig rasp-k8s-master-${instance}:~
  ssh rasp-k8s-master-${instance} "\
    sudo mv kube-scheduler.service /etc/systemd/system/
    sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
    sudo mv kube-scheduler.yaml /etc/kubernetes/config/
  "
done
rm kube-scheduler.service kube-scheduler.yaml
```

#### Start the Controller Services
```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    sudo systemctl daemon-reload
    sudo systemctl enable kube-scheduler
    sudo systemctl start kube-scheduler &
  "
done
```
```
for instance in 1; do
  ssh rasp-k8s-master-${instance} "\
    sudo systemctl status kube-scheduler -l
  "
done
```



### Enable HTTP Health Checks
```
sudo apt-get update
sudo apt-get install -y nginx
```

```
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```

```
sudo mv kubernetes.default.svc.cluster.local \
  /etc/nginx/sites-available/kubernetes.default.svc.cluster.local
sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
sudo systemctl restart nginx
sudo systemctl enable nginx
```
#### Verification
```
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```
#### Test the nginx HTTP health check proxy
```
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

### RBAC for Kubelet Authorization
```
ssh rasp-k8s-master-1
```
#### Create the ```system:kube-apiserver-to-kubelet```
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```
#### Bind the ```system:kube-apiserver-to-kubelet```
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

