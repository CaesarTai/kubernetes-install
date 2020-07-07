# k8s安装部署-kubeadm
 * 硬件环境
CPU i7-8750H 2.20GHz
RAM 16GB
 * 软件环境 
Ubuntu 16.04.3 LTS
## 环境准备
### 禁用swap
>    一般来说，Linux的虚拟内存会根据系统负载自动调整。内存页（page）swap到磁盘会显著的影响Kafka的性能，并且Kafka重度使用   page cache，如果VM系统swap到磁盘，那说明没有足够的内存来分配page cache。
```
$ swapoff -a           #临时关闭
$ vim /etc/fstab       #注释或删除掉swap那一整行，永久禁用
```
### 关闭防火墙
```
systemctl stop firewalld
systemctl disable firewalld
```
### 禁用SeLinux
```
apt install selinux-utils
setenforce 0
```
### 配置host
在每台虚拟机的/etc/hosts中，配置上集群所有机器的ip和主机名。如：
```
172.16.179.233 MasterName
172.16.179.234 Node-1
...
```
## 软件安装
### docker安装
在Ubuntu终端依次执行下面的语句，进行docker的安装
```
$ sudo apt install docker.io
$ sudo systemctl start docker
$ sudo systemctl enable docker
```
查看是否安装成功
```
$ docker -v
Docker version 19.03.6, build 369ce74a3c
```
### k8s安装(集群每台机器上执行)
添加 Kubernetes  signing key 和Repository
```
sudo apt install curl
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

安装`kubelet` 、`kubeadm`、`kubectl` ，`kubelet` 是 k8s 相关服务，`kubectl` 是 k8s 管理客户端，`kubeadm` 是部署工具。
```
apt-get install -y kubelet kubeadm kubectl --allow-unauthenticated
```
也可以指定版本号安装
```
apt update && apt-get install -y kubelet=1.13.1-00 kubernetes-cni=0.6.0-00 kubeadm=1.13.1-00 kubectl=1.13.1-00
```

## 集群配置
### 配置Master(只在Master执行)
在master节点 /etc/profile 下面增加如下环境变量
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
重启kubelet
```
systemctl daemon-reload 
systemctl restart kubelet
```
执行kubeinit
```
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=172.16.179.233 --kubernetes-version=v1.13.1
```
> 其中，`--pod-network-cidr`是规定pod内部可用的IP段，`--apiserver-advertise-address`是Master的IP，`--kubernetes-version`是当前k8s版本号。

安装成功后，会有如下提示。
![](https://github.com/CaesarTai/kubernetes-install/blob/master/pics/pic1.png)

需要执行图中提示的语句,将admin配置应用到集群配置中
```
mkdir -p $HOME/.kube 
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
> kubeadm init时候可能会出现一些镜像无法获取导致无法创建集群，可以手动将这些镜像拉取并且重新进行tag
> 可以用`mirrorgooglecontainers`来替换`k8s.gcr.io`进行安装
```
docker pull mirrorgooglecontainers/kube-apiserver-amd64:v1.13.1
docker pull mirrorgooglecontainers/kube-controller-manager-amd64:v1.13.1
docker pull mirrorgooglecontainers/kube-scheduler-amd64:v1.13.1
docker pull mirrorgooglecontainers/kube-proxy-amd64:v1.13.1
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/etcd-amd64:3.2.24
docker pull coredns/coredns:1.2.6
```
重新tag
> 即将从国内源下载的镜像标记，并归入k8s仓库
```
docker tag mirrorgooglecontainers/kube-controller-manager-amd64:v1.13.1 k8s.gcr.io/kube-controller-manager:v1.13.1
docker tag mirrorgooglecontainers/kube-scheduler-amd64:v1.13.1 k8s.gcr.io/kube-scheduler:v1.13.1
docker tag mirrorgooglecontainers/kube-apiserver-amd64:v1.13.1 k8s.gcr.io/kube-apiserver:v1.13.1
docker tag mirrorgooglecontainers/kube-proxy-amd64:v1.13.1 k8s.gcr.io/kube-proxy:v1.13.1
docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag mirrorgooglecontainers/etcd-amd64:3.2.24 k8s.gcr.io/etcd:3.2.24
docker tag coredns/coredns:1.2.6 k8s.gcr.io/coredns:1.2.6
```
> 更多国内源可参考：
https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/
https://blog.csdn.net/networken/article/details/84571373

### 配置Node
执行Master配置返回的join命令，加入该集群
```
kubeadm join 172.16.179.233:6443 --token 51eo0p.dpd3ffh9x27omcb2 --discovery-token-ca-cert-hash sha256:de66e223697e13c12ad977ab9a84fbbf83aea7bb903c7fea6fb2f198e88353af
```
![](https://github.com/CaesarTai/kubernetes-install/blob/master/pics/pic2.png)

### 安装配置网络插件
以flannel为例
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
等待下载完成会自动配置。

### 查看安装结果
输入`kubectl get nodes`，节点状态为ready则表示成功；
![](https://github.com/CaesarTai/kubernetes-install/blob/master/pics/pic3.png)

若不成功，可以通过`kubectl get pod --all-namespaces`检查是哪个pod出了问题。
![](https://github.com/CaesarTai/kubernetes-install/blob/master/pics/pic4.png)

