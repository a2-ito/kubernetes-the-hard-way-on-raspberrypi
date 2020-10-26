# 04 build

## Build etcd 
```
wget https://github.com/etcd-io/etcd/archive/v3.3.25.tar.gz
```

``` 
git clone https://github.com/etcd-io/etcd.git
export GOOS=linux
export GOARCH=arm
export GOARM=7
cd etcd/
./build
mv ./bin/* /build/
exit
``` 

``` 

``` 


##
```
wget https://github.com/kubernetes/kubernetes/archive/v1.16.2.tar.gz
wget https://dl.google.com/go/go1.12.12.linux-amd64.tar.gz
tar xzf go1.12.12.linux-amd64.tar.gz
sudo mv go /usr/local/
go version

tar xzf v1.16.2.tar.gz
cd kubernetes-1.16.2/
make
sudo apt install gcc-arm-linux-gnueabi
cp -p hack/lib/golang.sh hack/lib/golang.sh.org
vi hack/lib/golang.sh
make all WHAT=cmd/kube-proxy KUBE_VERBOSE=6 KUBE_BUILD_PLATFORMS=linux/arm
make all WHAT=cmd/kubelet KUBE_VERBOSE=6 KUBE_BUILD_PLATFORMS=linux/arm
make all WHAT=cmd/kubectl KUBE_VERBOSE=6 KUBE_BUILD_PLATFORMS=linux/arm
make all WHAT=cmd/kube-apiserver KUBE_VERBOSE=6 KUBE_BUILD_PLATFORMS=linux/arm
make all WHAT=cmd/kube-controller-manager KUBE_VERBOSE=6 KUBE_BUILD_PLATFORMS=linux/arm
make all WHAT=cmd/kube-scheduler KUBE_VERBOSE=6 KUBE_BUILD_PLATFORMS=linux/arm
```

## Copy
```
for instance in 1 2 3; do
  scp _output/local/bin/linux/arm/kube-apiserver rasp-master-${instance}:~
  scp _output/local/bin/linux/arm/kube-scheduler rasp-master-${instance}:~
  scp _output/local/bin/linux/arm/kube-controller-manager rasp-master-${instance}:~
  scp _output/local/bin/linux/arm/kubectl rasp-master-${instance}:~
done
```


