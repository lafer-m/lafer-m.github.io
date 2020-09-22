## kubeadm工具
k8s部署工具

## kubespray using kubeadm to bootstrap  kuberntes

### init first masters[0] node
```
command: >-
    timeout -k 300s 300s
    {{ bin_dir }}/kubeadm init
    --config={{ kube_config_dir }}/kubeadm-config.yaml
    --ignore-preflight-errors=all
    --skip-phases=addon/coredns
    --upload-certs
```

### create cert and key with ca
```
certsphase.CreateCertAndKeyFilesWithCA(cert, caCert, cfg)  // load ca from disk

certSpec.CreateFromCA(cfg, caCert, caKey) // create

pkiutil.NewCertAndKey(caCert, caKey, cfg) // new 

NewSignedCert(config, key, caCert, caKey) // new client cert

```
### 修改签发证书的默认有效期为10年
代码push到infra内部的代码分支上  
源码目录下  cmd/kubeadm/app/constants/constants.go
```
CertificateValidity = time.Hour * 24 * 365 * 10
```
### k8s build image use  build/run.sh 
以下操作在我们自己更新的kubernetes分支中操作
go/src/xxxx/kubernetes
```
k8s build 文档请参考源码目录下build/README.md
```

```
# 手工build 依赖的镜像，依赖的镜像没有拉的权限
cd kubernetes/build/build-image/cross
docker build -t staging-k8s.gcr.io/kube-cross:`cat VERSION` .
```

```
# build kubeadm 可执行文件
export KUBE_GIT_COMMIT=b31f398e7f046f917f78cceacf1cafbf5613afd9
export KUBE_GIT_VERSION=v1.15.9
export KUBE_GIT_MAJOR=1
export KUBE_GIT_MINOR=15
export KUBE_GIT_TREE_STATE=clean
export KUBE_BUILD_PLATFORMS="linux/amd64"
bash build/run.sh make kubeadm
```



### build the binary use make 
docker build ci 环境这个是自己的测试镜像
```
FROM centos:7

RUN yum install -y wget cmake make which rsync

RUN wget https://dl.google.com/go/go1.12.9.linux-amd64.tar.gz && \
    tar -C /usr/local -xzf go1.12.9.linux-amd64.tar.gz && \
    mkdir -p /go/{src,bin,pkg} && \
    chmod -R 777 /go

ENV GOPATH=/go \
    GO111MODULE=off \
    PATH=$PATH:/usr/local/go/bin
WORKDIR /go
```

build 需要填写版本号信息
```
export KUBE_GIT_COMMIT=b31f398e7f046f917f78cceacf1cafbf5613afd9
export KUBE_GIT_VERSION=v1.15.5
export KUBE_GIT_MAJOR=1
export KUBE_GIT_MINOR=15
export KUBE_GIT_TREE_STATE=clean
export KUBE_BUILD_PLATFORMS="linux/arm64"
make all WHAT=cmd/kubeadm GOFLAGS=-v
```




