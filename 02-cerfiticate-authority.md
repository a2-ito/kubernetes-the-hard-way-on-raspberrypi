# 02-Certificate-authoriry

## Create Authority
```
mkdir ./keys
```
```
cat > ./keys/ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ./keys/ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Yokohama",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Kanagawa"
    }
  ]
}
EOF
```

```
cfssl gencert -initca ./keys/ca-csr.json | cfssljson -bare ca
mv ./*.pem ./*.csr ./keys/
```
## Create Certificates
```
cat > ./keys/template-csr.json <<EOF
{
  "CN": "1_CN",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "2_C",
      "L": "3_L",
      "O": "4_O",
      "OU": "5_OU",
      "ST": "6_ST"
    }
  ]
}
EOF
```
```
_C="JP"
_L="Yokohama"
_ST="Kanagawa"
sbj=(
admin';'admin';'${_C}';'${_L}';'system:masters';'"Kubernetes The Hard Way"';'${_ST}';'""
kubernetes';'kubernetes';'${_C}';'${_L}';'Kubernetes';'"Kubernetes The Hard Way"';'${_ST}';'"10.32.0.1,192.168.11.11,192.168.11.12,192.168.11.13,192.168.11.14,127.0.0.1,rasp-etcd-1,rasp-etcd-2,rasp-etcd-3,rasp-k8s-master-1"
kube-controller-manager';'system:kube-controller-manager';'${_C}';'${_L}';'system:kube-controller-manager';'"Kubernetes The Hard Way"';'${_ST}';'""
kube-scheduler';'system:kube-scheduler';'${_C}';'${_L}';'system:kube-scheduler';'"Kubernetes The Hard Way"';'${_ST}';'""
rasp-worker-1';'system:node:rasp-worker-1';'${_C}';'${_L}';'system:nodes';'"Kubernetes The Hard Way"';'${_ST}';'"rasp-worker-1,192.168.11.21"
rasp-worker-2';'system:node:rasp-worker-2';'${_C}';'${_L}';'system:nodes';'"Kubernetes The Hard Way"';'${_ST}';'"rasp-worker-2,192.168.11.22"
rasp-worker-3';'system:node:rasp-worker-3';'${_C}';'${_L}';'system:nodes';'"Kubernetes The Hard Way"';'${_ST}';'"rasp-worker-3,192.168.11.23"
kube-proxy';'system:kube-proxy';'${_C}';'${_L}';'system:node-proxier';'"Kubernetes The Hard Way"';'${_ST}';'""
service-account';'service-accounts';'${_C}';'${_L}';'Kubernetes';'"Kubernetes The Hard Way"';'${_ST}';'""
)
for array in "${sbj[@]}"
do
  servertype=`echo "${array}" | cut -d';' -f1`
  _1_CN=`echo "${array}" | cut -d';' -f2`
  _2_C=`echo "${array}" | cut -d';' -f3`
  _3_L=`echo "${array}" | cut -d';' -f4`
  _4_O=`echo "${array}" | cut -d';' -f5`
  _5_OU=`echo "${array}" | cut -d';' -f6`
  _6_ST=`echo "${array}" | cut -d';' -f7`
  _7_HN=`echo "${array}" | cut -d';' -f8`
  echo ${_6_ST}
  cat ./keys/template-csr.json |\
  sed -e "s/1_CN/${_1_CN}/g" \
	-e "s/2_C/${_2_C}/g" \
	-e "s/3_L/${_3_L}/g" \
	-e "s/4_O/${_4_O}/g" \
	-e "s/5_OU/${_5_OU}/g" \
	-e "s/6_ST/${_6_ST}/g" > ./keys/${servertype}-csr.json
  cfssl gencert \
    -ca=./keys/ca.pem \
    -ca-key=./keys/ca-key.pem \
    -config=./keys/ca-config.json \
    -profile=kubernetes \
    -hostname="${_7_HN}" \
    ./keys/${servertype}-csr.json | cfssljson -bare ${servertype}
done
mv ./*.pem ./*.csr ./keys/
```

```
for instance in 1 2 3;
do
  scp ./keys/ca.pem ./keys/ca-key.pem ./keys/kubernetes-key.pem ./keys/kubernetes.pem \
      rasp-etcd-${instance}:~
done
```

```
for instance in 1;
do
  scp ./keys/ca.pem ./keys/ca-key.pem ./keys/kubernetes-key.pem ./keys/kubernetes.pem \
      ./keys/service-account-key.pem ./keys/service-account.pem rasp-k8s-master-${instance}:~
done
```

