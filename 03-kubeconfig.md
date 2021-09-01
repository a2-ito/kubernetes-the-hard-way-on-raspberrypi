# 03-kubeconfig
## The kubelet Kubernetes Configuration File
```
for servertype in rasp-worker-1 rasp-worker-2 rasp-worker-3; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=./keys/ca.pem \
    --embed-certs=true \
    --server=https://192.168.11.14:6443 \
    --kubeconfig=${servertype}.kubeconfig

  kubectl config set-credentials system:node:${servertype} \
    --client-certificate=./keys/${servertype}.pem \
    --client-key=./keys/${servertype}-key.pem \
    --embed-certs=true \
    --kubeconfig=${servertype}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${servertype} \
    --kubeconfig=${servertype}.kubeconfig

  kubectl config use-context default --kubeconfig=${servertype}.kubeconfig
done
```

## The kube-proxy Kubernetes Configuration File
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=./keys/ca.pem \
  --embed-certs=true \
  --server=https://192.168.11.14:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=./keys/kube-proxy.pem \
  --client-key=./keys/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

## The kube-controller-manager Kubernetes Configuration File
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=./keys/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=./keys/kube-controller-manager.pem \
  --client-key=./keys/kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

## The kube-scheduler Kubernetes Configuration File
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=./keys/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=./keys/kube-scheduler.pem \
  --client-key=./keys/kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

## The admin Kubernetes Configuration File
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=./keys/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=./keys/admin.pem \
  --client-key=./keys/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

## Copy Configuration Files to Master
```
for instance in 1; do
  scp ./kube-controller-manager.kubeconfig \
      ./kube-scheduler.kubeconfig \
      ./admin.kubeconfig \
      rasp-k8s-master-${instance}:~
done
```

## Copy Configuration Files to Worker
```
for instance in 1 2 3; do
  scp ./kube-proxy.kubeconfig ./admin.kubeconfig rasp-worker-${instance}:~
done
```

