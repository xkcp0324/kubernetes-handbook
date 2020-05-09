# 国内镜像列表

## Docker Hub 镜像

- 中科大镜像：`docker.mirrors.ustc.edu.cn`
- 163镜像： `hub-mirror.c.163.com`
- 开源社镜像：<http://mirror.azure.cn/help/docker-registry-proxy-cache.html>

示例

```sh
docker pull docker.mirrors.ustc.edu.cn/library/nginx
docker pull hub-mirror.c.163.com/library/busybox
docker pull dockerhub.azk8s.cn/library/nginx

docker pull quay.mirrors.ustc.edu.cn/coreos/kube-state-metrics:v1.5.0
```

## GCR（Google Container Registry）镜像

- 开源社镜像（Azure China）：http://mirror.azure.cn/help/gcr-proxy-cache.html
- 阿里云镜像：registry.cn-hangzhou.aliyuncs.com/google_containers 或者 registry.aliyuncs.com/google_containers

示例

```sh
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.18.0
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.18.0

docker pull gcr.azk8s.cn/google_containers/hyperkube:v1.12.1
docker pull gcr.azk8s.cn/google_containers/pause-amd64:3.1
```

## Kubernetes RPM/DEB镜像

- [开源社镜像](http://mirror.azure.cn/kubernetes/packages/)

示例：

```sh
# Ubuntu
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://mirror.azure.cn/kubernetes/packages/apt/ kubernetes-xenial main
EOF
```

### Helm Charts 镜像

- Helm: http://mirror.azure.cn/kubernetes/helm/
- Stable Charts: http://mirror.azure.cn/kubernetes/charts/
- Incubator Charts: http://mirror.azure.cn/kubernetes/charts-incubator/

示例

```sh
helm repo add stable http://mirror.azure.cn/kubernetes/charts/
helm repo add incubator http://mirror.azure.cn/kubernetes/charts-incubator/
```

## 操作系统镜像

- [开源社开源镜像](http://mirror.azure.cn/)
- [网易开源镜像](https://mirrors.163.com/)

以 Ubuntu 18.04（Bionic）为例，修改 /etc/apt/sources.list 文件的内容为

```sh
deb http://azure.archive.ubuntu.com/ubuntu/ bionic main restricted universe multiverse
deb http://azure.archive.ubuntu.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://azure.archive.ubuntu.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://azure.archive.ubuntu.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://azure.archive.ubuntu.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://azure.archive.ubuntu.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://azure.archive.ubuntu.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://azure.archive.ubuntu.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://azure.archive.ubuntu.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://azure.archive.ubuntu.com/ubuntu/ bionic-backports main restricted universe multiverse
```
