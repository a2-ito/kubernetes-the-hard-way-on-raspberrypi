## 08 kubeconfig

```
kubectl config set-cluster rasp-cluster --server=https://192.168.11.14:6443 --certificate-authority=./keys/ca.pem
kubectl config set-credentials rasp-cluster --client-certificate=./keys/admin.pem --client-key=./keys/admin-key.pem
kubectl config set-context rasp-cluster --user=rasp-cluster --cluster=rasp-cluster
kubectl config use-context rasp-cluster
kubectl config view
```

```
kubectl get node
kubectl get pod -A
```
