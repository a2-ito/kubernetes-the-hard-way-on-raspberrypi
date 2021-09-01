## 11-smoke-test

```
kubectl create deployment nginx --image=nginx
kubectl scale --replicas=3 deployment/nginx 
```

```
for instance in 1 2 3; do
  for i in `seq 0 2`; do
    POD_IP=$(kubectl get pods -o jsonpath="{.items["${i}"].status.podIP}")
    _pod=`kubectl get pod -o wide | grep rasp-worker-${instance} | cut -f1 -d' '`
    echo ${_pod} on rasp-worker-${instance}" => "${POD_IP}
    kubectl exec -it ${_pod} \
      -- sh -c "curl -s --head http://${POD_IP}"
    echo;
    sleep 5
  done
done
```
```
kubectl delete deploy nginx
```
