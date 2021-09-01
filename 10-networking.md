# 10 Networking

```
for instance in 1 2 3
do
  kubectl run --image=busybox --restart=Never busybox-${instance} \
    --overrides='{"spec": { "nodeSelector": {"kubernetes.io/hostname": "rasp-worker-'${instance}'"}}}' \
    -- sleep 3600
done
```

## Option 1. CNI
```
for instance in 1 2 3
do
  case "${instance}" in
  1)
  ssh rasp-worker-${instance} "\
    sudo sh -c \"sudo ip route add 10.200.2.0/24 via 192.168.11.22 dev eth0\"
    sudo sh -c \"sudo ip route add 10.200.3.0/24 via 192.168.11.23 dev eth0\"
  "
  ;;
  2)
  ssh rasp-worker-${instance} "\
    sudo sh -c \"sudo ip route add 10.200.1.0/24 via 192.168.11.21 dev eth0\"
    sudo sh -c \"sudo ip route add 10.200.3.0/24 via 192.168.11.23 dev eth0\"
  "
  ;;
  3)
  ssh rasp-worker-${instance} "\
    sudo sh -c \"sudo ip route add 10.200.1.0/24 via 192.168.11.21 dev eth0\"
    sudo sh -c \"sudo ip route add 10.200.2.0/24 via 192.168.11.22 dev eth0\"
  "
  ;;
  *)
    ;;
  esac
done
```
```
for instance in 1 2 3
do
  ssh rasp-worker-${instance} "\
    sudo ip route
  "
done
```


### Option 2. flannel
```
for instance in 1 2 3
do
  ssh rasp-worker-${instance} "\
  sudo sysctl net.ipv4.conf.all.forwarding=1
  sudo sh -c 'echo \"net.ipv4.conf.all.forwarding = 1\" >> /etc/sysctl.conf'
  "
done
```

```
POD_CIDR=10.200.0.0\\\/16
cp -p ./manifests/kube-flannel.yml .
sed -i s/POD_CIDR/${POD_CIDR}/g kube-flannel.yml
kubectl apply -f ./manifests/kube-flannel.yml
```

## Check connectivity
```
for instance in 1 2 3
do
  kubectl get pods busybox-${instance} -o jsonpath='{.status.podIP}'
  echo;
done
```

Then, you should check a connectivity between all pods on each pod.
```
ping -c5 10.200.2.4
```

