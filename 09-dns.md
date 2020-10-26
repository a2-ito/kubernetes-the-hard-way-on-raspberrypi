# 08 Deploying the DNS Cluster Add-on

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
```
### Verification
```
kubectl run --image=busybox:1.28 --rm --restart=Never -i testpod \
  --overrides='{"spec": { "nodeSelector": {"kubernetes.io/hostname": "rasp-worker-1"}}}' \
  -- nslookup kubernetes
kubectl run --image=busybox:1.28 --rm --restart=Never -i testpod \
  --overrides='{"spec": { "nodeSelector": {"kubernetes.io/hostname": "rasp-worker-2"}}}' \
  -- nslookup kubernetes
kubectl run --image=busybox:1.28 --rm --restart=Never -i testpod \
  --overrides='{"spec": { "nodeSelector": {"kubernetes.io/hostname": "rasp-worker-3"}}}' \
  -- nslookup kubernetes
```

```
kubectl delete deployment coredns -n kube-system
kubectl delete svc kube-dns -n kube-system
```

