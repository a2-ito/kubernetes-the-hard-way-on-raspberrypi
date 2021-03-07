# Ingree

## Deploy Traefik

```
kubectl create -f manifests/configmap-traefik.yaml
kubectl create -f manifests/deploy-traefik.yaml
kubectl create -f manifests/svc-traefik.yaml
```

## Configure Ingress with Traefik

```
kubectl create -f manifests/ingress-dashboard.yaml
```

```
kubectl patch svc svc-traefik-ingress \
  -p '{"spec": {"type": "LoadBalancer", "externalIPs":["192.168.11.21"]}}' -n kube-system
```

